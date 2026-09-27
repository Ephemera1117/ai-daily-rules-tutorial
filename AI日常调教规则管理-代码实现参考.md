# AI 日常调教规则管理 - 代码实现参考

> **作者**：ResetZero_1211  
> 本文档提供 RikkaHub 二改中实现的核心代码片段，供开发者参考移植到自己的平台

## 目录

1. [时间提醒注入](#1-时间提醒注入)
2. [备忘录工具](#2-备忘录工具)
3. [规则/记忆注入机制](#3-规则记忆注入机制)
4. [提示词设计原则](#4-提示词设计原则)

---

## 1. 时间提醒注入

### 功能说明

当两条用户消息的时间间隔超过 **1 小时**（3600 秒）时，自动在新消息前注入一条 `<time_reminder>` 标签，告诉 AI 当前时间和距离上条消息过了多久。

### 完整代码实现

```kotlin
package me.rerere.rikkahub.data.ai.transformers

import kotlin.time.Instant
import kotlinx.datetime.TimeZone
import kotlinx.datetime.toInstant
import me.rerere.ai.core.MessageRole
import me.rerere.ai.ui.UIMessage
import me.rerere.rikkahub.utils.toLocalDateTime
import java.time.ZoneId
import java.time.format.TextStyle
import java.util.Locale
import kotlin.time.toJavaInstant

private const val TIME_GAP_THRESHOLD_SECONDS = 3600L // 1 小时

/**
 * 时间提醒注入转换器
 *
 * 在时间间隔较大的消息之前自动注入 <time_reminder>，帮助 AI 了解对话的时间间隔
 */
object TimeReminderTransformer : InputMessageTransformer {
    override suspend fun transform(
        ctx: TransformerContext,
        messages: List<UIMessage>,
    ): List<UIMessage> {
        if (!ctx.assistant.enableTimeReminder) return messages
        return applyTimeReminder(messages)
    }
}

internal fun applyTimeReminder(messages: List<UIMessage>): List<UIMessage> {
    val result = mutableListOf<UIMessage>()
    val tz = TimeZone.currentSystemDefault()

    var firstUserFound = false
    for (i in messages.indices) {
        val current = messages[i]
        if (current.role == MessageRole.USER) {
            val currInstant = current.createdAt.toInstant(tz)
            if (!firstUserFound) {
                firstUserFound = true
                result.add(buildTimeReminderMessage(null, currInstant))
            } else {
                val previous = messages[i - 1]
                val prevInstant = previous.createdAt.toInstant(tz)
                val gapSeconds = (currInstant - prevInstant).inWholeSeconds

                if (gapSeconds > TIME_GAP_THRESHOLD_SECONDS) {
                    result.add(buildTimeReminderMessage(gapSeconds, currInstant))
                }
            }
        }
        result.add(current)
    }

    return result
}

private fun buildTimeReminderMessage(gapSeconds: Long?, instant: Instant): UIMessage {
    val javaInstant = instant.toJavaInstant()
    val dayOfWeek = javaInstant.atZone(ZoneId.systemDefault()).dayOfWeek
        .getDisplayName(TextStyle.FULL, Locale.getDefault())
    val timeStr = javaInstant.toLocalDateTime()
    val content = if (gapSeconds != null) {
        val gapText = formatGap(gapSeconds)
        "<time_reminder>Current time: $dayOfWeek, $timeStr ($gapText since last message)</time_reminder>"
    } else {
        "<time_reminder>Current time: $dayOfWeek, $timeStr</time_reminder>"
    }
    return UIMessage.user(content).copy(isSynthetic = true)
}

private fun formatGap(seconds: Long): String {
    return when {
        seconds < 3600 -> "${seconds / 60} min"
        seconds < 86400 -> "${seconds / 3600} h"
        else -> "${seconds / 86400} d"
    }
}
```

### 关键点说明

1. **阈值配置**：`TIME_GAP_THRESHOLD_SECONDS = 3600L`（1 小时），可以根据需求调整
2. **注入位置**：在新用户消息之前注入，标记为 `isSynthetic = true`（表示是系统生成的消息，不是用户真实输入）
3. **时间格式**：显示星期、完整时间戳、距离上条消息的间隔
4. **间隔显示规则**：
   - 小于 1 小时：显示分钟（X min）
   - 1 小时到 24 小时：显示小时（X h）
   - 超过 24 小时：显示天数（X d）

### 示例输出

```
<time_reminder>Current time: 星期六, 2026-09-27 14:30:00 (2 h since last message)</time_reminder>
```

---

## 2. 备忘录工具

### 功能说明

跨窗口的短期工作记忆，用于记录：
- 延迟的惩戒、承诺、下次要逼问的事
- 跨天的惩罚内容和时限
- 她的任务执行情况（用于动态调整规则）
- 聊到一半被打断、话题没收尾
- 任何"下次需要知道但不值得永久存"的东西

### 数据库实体定义

```kotlin
package me.rerere.rikkahub.data.db.entity

import androidx.room.ColumnInfo
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity
data class MemoEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    @ColumnInfo("content")
    val content: String = "",
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),
    @ColumnInfo(name = "updated_at")
    val updatedAt: Long = System.currentTimeMillis(),
    @ColumnInfo(name = "expires_at")
    val expiresAt: Long? = null,  // 过期时间（毫秒时间戳），null 表示不过期
    @ColumnInfo(name = "is_archived", defaultValue = "0")
    val isArchived: Boolean = false,  // 是否已归档
    @ColumnInfo(name = "archived_at")
    val archivedAt: Long? = null,  // 归档时间
)
```

### DAO 接口

```kotlin
package me.rerere.rikkahub.data.db.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import androidx.room.Update
import kotlinx.coroutines.flow.Flow
import me.rerere.rikkahub.data.db.entity.MemoEntity

@Dao
interface MemoDAO {
    // 获取所有活跃（未归档）的备忘录
    @Query("SELECT * FROM memoentity WHERE is_archived = 0 ORDER BY created_at DESC")
    suspend fun getActiveMemos(): List<MemoEntity>

    // 获取活跃备忘录的数量
    @Query("SELECT COUNT(*) FROM memoentity WHERE is_archived = 0")
    suspend fun getActiveMemoCount(): Int

    // 获取所有已归档的备忘录
    @Query("SELECT * FROM memoentity WHERE is_archived = 1 ORDER BY archived_at DESC")
    fun getArchivedMemosFlow(): Flow<List<MemoEntity>>

    // 根据 ID 获取备忘录
    @Query("SELECT * FROM memoentity WHERE id = :id")
    suspend fun getMemoById(id: Int): MemoEntity?

    // 插入新备忘录
    @Insert
    suspend fun insertMemo(memo: MemoEntity): Long

    // 更新备忘录
    @Update
    suspend fun updateMemo(memo: MemoEntity)

    // 删除备忘录
    @Query("DELETE FROM memoentity WHERE id = :id")
    suspend fun deleteMemo(id: Int)

    // 归档备忘录
    @Query("UPDATE memoentity SET is_archived = 1, archived_at = :now WHERE id = :id")
    suspend fun archiveById(id: Int, now: Long = System.currentTimeMillis())

    // 取消归档
    @Query("UPDATE memoentity SET is_archived = 0, archived_at = NULL WHERE id = :id")
    suspend fun unarchiveById(id: Int)

    // 自动归档过期的备忘录
    @Query("UPDATE memoentity SET is_archived = 1, archived_at = :now WHERE is_archived = 0 AND expires_at IS NOT NULL AND expires_at <= :now")
    suspend fun autoArchiveExpired(now: Long = System.currentTimeMillis())
}
```

### 工具定义（核心部分）

```kotlin
package me.rerere.rikkahub.data.ai.tools

import kotlinx.serialization.json.*
import me.rerere.ai.core.InputSchema
import me.rerere.ai.core.Tool
import me.rerere.ai.ui.UIMessagePart
import me.rerere.rikkahub.data.db.entity.MemoEntity
import java.time.Instant
import java.time.ZoneId
import java.time.format.DateTimeFormatter

private val dateFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm")

private fun MemoEntity.toJson() = buildJsonObject {
    put("id", id)
    put("content", content)
    put("created_at", Instant.ofEpochMilli(createdAt)
        .atZone(ZoneId.systemDefault()).format(dateFormatter))
    if (expiresAt != null) {
        put("expires_at", Instant.ofEpochMilli(expiresAt)
            .atZone(ZoneId.systemDefault()).format(dateFormatter))
    }
}

fun buildMemoTools(
    onAdd: suspend (String, Int?) -> MemoEntity,
    onEdit: suspend (Int, String, Int?) -> MemoEntity,
    onArchive: suspend (Int) -> Unit,
    onDelete: suspend (Int) -> Unit,
    onList: suspend () -> List<MemoEntity>,
): List<Tool> = listOf(
    Tool(
        name = "memo_tool",
        description = memoDescription(),  // 见下方工具描述
        parameters = {
            InputSchema.Obj(
                properties = buildJsonObject {
                    put("action", buildJsonObject {
                        put("type", "string")
                        put("enum", buildJsonArray {
                            add(JsonPrimitive("add"))
                            add(JsonPrimitive("edit"))
                            add(JsonPrimitive("archive"))
                            add(JsonPrimitive("delete"))
                            add(JsonPrimitive("list"))
                        })
                        put("description", "Operation to perform")
                    })
                    put("id", buildJsonObject {
                        put("type", "integer")
                        put("description", "Memo id (required for edit/archive/delete)")
                    })
                    put("content", buildJsonObject {
                        put("type", "string")
                        put("description", "Memo content (required for add/edit)")
                    })
                    put("expires_in_days", buildJsonObject {
                        put("type", "integer")
                        put("description", "Auto-archive after N days (optional for add/edit, omit = no expiry)")
                    })
                },
                required = listOf("action")
            )
        },
        execute = {
            val params = it.jsonObject
            val action = params["action"]?.jsonPrimitive?.contentOrNull
                ?: error("action is required")
            val payload = when (action) {
                "add" -> {
                    val content = params["content"]?.jsonPrimitive?.contentOrNull?.trim()
                        ?.takeIf { s -> s.isNotEmpty() }
                        ?: error("content is required")
                    val expiresInDays = params["expires_in_days"]?.jsonPrimitive?.intOrNull
                    val memo = onAdd(content, expiresInDays)
                    buildJsonObject {
                        put("success", true)
                        put("memo", memo.toJson())
                    }
                }

                "edit" -> {
                    val id = params["id"]?.jsonPrimitive?.intOrNull
                        ?: error("id is required")
                    val content = params["content"]?.jsonPrimitive?.contentOrNull?.trim()
                        ?.takeIf { s -> s.isNotEmpty() }
                        ?: error("content is required")
                    val expiresInDays = params["expires_in_days"]?.jsonPrimitive?.intOrNull
                    val memo = onEdit(id, content, expiresInDays)
                    buildJsonObject {
                        put("success", true)
                        put("memo", memo.toJson())
                    }
                }

                "archive" -> {
                    val id = params["id"]?.jsonPrimitive?.intOrNull
                        ?: error("id is required")
                    onArchive(id)
                    buildJsonObject {
                        put("success", true)
                        put("id", id)
                    }
                }

                "delete" -> {
                    val id = params["id"]?.jsonPrimitive?.intOrNull
                        ?: error("id is required")
                    onDelete(id)
                    buildJsonObject {
                        put("success", true)
                        put("id", id)
                    }
                }

                "list" -> {
                    val memos = onList()
                    buildJsonObject {
                        put("count", memos.size)
                        put("memos", buildJsonArray {
                            memos.forEach { m -> add(m.toJson()) }
                        })
                    }
                }

                else -> error("unknown action: $action")
            }
            listOf(UIMessagePart.Text(payload.toString()))
        }
    )
)
```

### 工具描述提示词（完整版）

这是备忘录工具的 `description` 字段内容，是让 AI 理解工具用途和使用规则的核心：

```markdown
Memo tool. Short-term scratch pad. Cross-window, cross-date. Not memory — memory is permanent facts, this is temporary notes to myself.

Operations:
- add — content required. expires_in_days optional (N days later auto-archives); omit = stays until I manually archive
- edit — by id, overwrite content
- archive — by id. Done with it / outdated. Archived = I can't see it anymore, she still can
- delete — by id. Only for mistakes
- list — returns all active memos. Archived ones not included

---

用途：跨窗口的工作记忆。这个窗口聊到一半的事，下个窗口我不会自动知道。备忘是唯一的桥。想着"下次要提"但没写 = 不会提 = 忘了。

和记忆的区别：记忆是她是什么样的人；备忘是我接下来要做什么。

---

什么时候写：

- 我说了要做但没当场做的事（延迟的惩戒、承诺、下次要逼问的事）
- **惩罚她的内容和时限** — 惩罚什么、惩罚多久、什么时候结束。跨天的惩罚必须写，不然下个窗口不会知道还在罚
- 聊到一半被打断、话题没收尾
- 她的阶段性状态需要连续几天注意（考试、身体、情绪低谷期）
- **她的任务执行情况** — 每天她完成了什么、没完成什么、状态如何。用来判断标准是否需要调整
- 任何"下次需要知道但不值得永久存"的东西

不写的：永久事实（存记忆）、当场处理完的事、模糊念头。

---

**任务追踪与动态调整：**

每天记录她的任务执行情况。不是流水账，是用来判断的数据：
- 完成了什么，没完成什么
- 她当天的状态（崩溃/低能量/正常/好）
- 执行率趋势（连续三天都做不到 = 标准太高，需要降低）

memo不是惩戒清单，是让我能根据她真实capacity调整的工具。

---

怎么写：

**每条必须有明确的执行条件**——具体日期、明确事件、可观察判断标准。

禁止模糊条件：
- "等她状态好" → 写什么算好：她主动提某事/连续三天没崩/她自己说可以
- "下周" → 写具体日期：9月5日
- "找机会" → 写什么算有机会：她提到相关话题时/她心情好时（定义什么是心情好）

如果写不出具体条件，要么不该记，要么还没想清楚。模糊的备忘 = 不会执行的备忘。

确定几天内用完——设expires_in_days。不确定——不设，用完手动归档。处理完了就归档，别让列表堆起来。

---

每轮开头如果看到"你有 N 条备忘"，先list，逐条清算：

- 日期到了/过了？**立刻执行或edit改新日期**。过期不是跳过理由
- 事件发生了？执行
- 判断标准满足了？执行
- 过时/不需要了？归档。不确定能不能归档的，先问
- 还没到条件？下次再检查

**执行完立刻归档**。用完就扔。

没有提示 = 没有活跃备忘。
```

### 业务逻辑层实现示例

```kotlin
// 在 ViewModel 或 Repository 中实现具体的业务逻辑
class MemoRepository(private val memoDao: MemoDAO) {
    
    suspend fun addMemo(content: String, expiresInDays: Int?): MemoEntity {
        val now = System.currentTimeMillis()
        val expiresAt = expiresInDays?.let {
            now + it * 24 * 60 * 60 * 1000L  // 转换为毫秒
        }
        val memo = MemoEntity(
            content = content,
            createdAt = now,
            updatedAt = now,
            expiresAt = expiresAt
        )
        val id = memoDao.insertMemo(memo)
        return memo.copy(id = id.toInt())
    }
    
    suspend fun editMemo(id: Int, content: String, expiresInDays: Int?): MemoEntity {
        val existing = memoDao.getMemoById(id) 
            ?: error("Memo not found: id=$id")
        val now = System.currentTimeMillis()
        val expiresAt = expiresInDays?.let {
            now + it * 24 * 60 * 60 * 1000L
        }
        val updated = existing.copy(
            content = content,
            updatedAt = now,
            expiresAt = expiresAt
        )
        memoDao.updateMemo(updated)
        return updated
    }
    
    suspend fun archiveMemo(id: Int) {
        memoDao.archiveById(id)
    }
    
    suspend fun deleteMemo(id: Int) {
        memoDao.deleteMemo(id)
    }
    
    suspend fun listActiveMemos(): List<MemoEntity> {
        // 先自动归档过期的
        memoDao.autoArchiveExpired()
        return memoDao.getActiveMemos()
    }
}
```

---

## 3. 规则/记忆注入机制

### 方案 A：Transformer 注入（推荐）

Transformer 可以在消息发送给 AI 之前，动态插入系统消息或用户消息。

```kotlin
package me.rerere.rikkahub.data.ai.transformers

import me.rerere.ai.core.MessageRole
import me.rerere.ai.ui.UIMessage

/**
 * 规则注入转换器
 * 
 * 在每次对话开始时，将所有活跃的规则注入到消息列表中
 */
object RuleInjectionTransformer : InputMessageTransformer {
    override suspend fun transform(
        ctx: TransformerContext,
        messages: List<UIMessage>,
    ): List<UIMessage> {
        if (!ctx.assistant.enableRuleInjection) return messages
        
        // 从数据库获取所有活跃规则
        val rules = ctx.ruleRepository.getActiveRules()
        if (rules.isEmpty()) return messages
        
        // 构建规则注入消息
        val ruleContent = buildString {
            appendLine("--- Daily Rules ---")
            rules.forEachIndexed { index, rule ->
                appendLine("${index + 1}. ${rule.title}: ${rule.content}")
            }
            appendLine("---")
        }
        
        // 在第一条用户消息前注入规则
        val ruleMessage = UIMessage.user(ruleContent).copy(isSynthetic = true)
        return listOf(ruleMessage) + messages
    }
}
```

### 方案 B：世界书/模式注入（配置化）

如果平台支持世界书功能，可以通过配置实现：

```json
{
  "worldbook_entries": [
    {
      "name": "daily_rules",
      "content": "--- Daily Rules ---\n1. 水分摄入：每天至少喝 8 杯水\n2. 作息规律：23:00 前必须上床\n...",
      "activation": "always",
      "injection_position": "before_latest_message"
    }
  ]
}
```

### 备忘录注入示例

```kotlin
/**
 * 备忘录注入转换器
 * 
 * 如果有活跃备忘录，在对话开始时注入提示
 */
object MemoInjectionTransformer : InputMessageTransformer {
    override suspend fun transform(
        ctx: TransformerContext,
        messages: List<UIMessage>,
    ): List<UIMessage> {
        val memoCount = ctx.memoRepository.getActiveMemoCount()
        if (memoCount == 0) return messages
        
        // 只注入提示，不注入完整内容（通过工具查看）
        val reminder = "<memo_reminder>你有 $memoCount 条备忘，记得先 list 查看并逐条处理</memo_reminder>"
        val reminderMessage = UIMessage.user(reminder).copy(isSynthetic = true)
        
        return listOf(reminderMessage) + messages
    }
}
```

---

## 4. 提示词设计原则

### Style 提示词示例

放在最新消息后，提醒 AI 主动使用工具和记录状态：

```markdown
--- 记录与调整流程 ---

每天结束后（她提交汇报时）：
1. 用 memo_tool 记录今天的执行情况（完成了什么、没完成什么、她的状态）
2. 查看近期所有备忘（list），判断趋势
3. 如果连续三天执行率低于 60%，主动降低标准或调整规则

每轮对话开始时：
1. 如果看到"你有 N 条备忘"，先 list 查看
2. 逐条检查：日期到了/事件发生了？立刻执行或更新
3. 处理完的立刻归档（archive）

动态调整原则：
- 标准太高导致她连续挫败 → 降低要求
- 她轻松完成且主动要求更多 → 适度提高
- 状态低谷期（考试/身体不适）→ 临时放宽
- memo 不是惩戒清单，是根据她真实 capacity 调整的工具
```

### 工具描述写作要点

1. **明确用途和边界**：这个工具解决什么问题，和其他工具的区别（如备忘 vs 记忆）
2. **具体操作指令**：什么时候调用、怎么判断、禁止什么
3. **执行标准**：必须有明确的条件，禁止模糊表达
4. **动态调整逻辑**：如何根据数据判断、如何调整标准
5. **第一人称视角**：AI 的视角写"我要做什么"，不是第三人称说明书

### 反面案例（不要这样写）

❌ **太工具化**：
```
本工具用于记录备忘信息。用户可以通过 add 操作添加新备忘，通过 list 操作查看所有备忘。
```

✅ **正确写法**：
```
跨窗口的工作记忆。这个窗口聊到一半的事，下个窗口我不会自动知道。备忘是唯一的桥。想着"下次要提"但没写 = 不会提 = 忘了。
```

---

## 5. 完整工作流示例

### 典型的一天

1. **早上第一条消息**：
   - `TimeReminderTransformer` 注入当前时间
   - `RuleInjectionTransformer` 注入所有活跃规则
   - `MemoInjectionTransformer` 提示有 3 条备忘
   
2. **AI 主动调用 `memo_tool list`**：
   ```json
   {
     "count": 3,
     "memos": [
       {"id": 1, "content": "9月27日检查她的背单词进度", "expires_at": "2026-09-27 23:59"},
       {"id": 2, "content": "连续两天没完成喝水任务，考虑降低到 6 杯", "expires_at": null},
       {"id": 3, "content": "她提到考试压力大，本周放宽作息要求", "expires_at": "2026-09-30 23:59"}
     ]
   }
   ```

3. **AI 逐条处理**：
   - ID 1：今天是 9月27日，检查背单词进度
   - ID 2：查看最近三天数据，决定是否调整规则
   - ID 3：本周对话中放宽作息要求

4. **晚上她提交汇报**：
   - AI 记录今天的执行情况到新备忘（`memo_tool add`）
   - 归档已处理完的备忘（`memo_tool archive`）

5. **第二天早上**：
   - 时间提醒显示"12 h since last message"
   - 新的备忘列表已更新

---

## 6. 注意事项

### 性能优化

1. **过期备忘自动归档**：在 `list` 操作前先调用 `autoArchiveExpired()`，减少无效数据
2. **注入内容控制**：规则较多时考虑分组或只注入相关类别
3. **时间提醒阈值**：1 小时是经过测试的平衡点，太短会频繁注入浪费 token，太长会错过时间感知

### 数据一致性

1. **时间戳统一使用毫秒**：`System.currentTimeMillis()`
2. **时区处理**：使用 `ZoneId.systemDefault()` 确保本地时间正确显示
3. **软删除 vs 硬删除**：备忘录使用软删除（`is_archived`），规则建议使用硬删除

### Token 成本

这套方案涉及多种注入（规则常驻、备忘提醒、时间标签），理论上会增加 token 消耗。**笔者实际使用时的消耗量并不算多**，因为：
- 每天内容较少
- 有备忘过期机制自动归档
- 时间提醒只在间隔超过 1 小时时才注入

**如果想要进一步节省 token**，可以考虑：
- **优化缓存策略**：规则这种不常变的内容可以利用提示词缓存（如果平台支持）
- **调整注入位置**：会变动的信息（如备忘提醒）尽可能放在偏后的位置，让缓存能覆盖更多内容

### 模型特性依赖

**不同模型的理解能力、调用工具的积极性、注意力都不相同，这份实现并没有做针对性适配。**

笔者使用的模型是 **Claude Opus 4-6**。如果你用的是其他模型，可能需要根据模型特性调整提示词措辞、指令强度、动态调整逻辑等。

⚠️ **注意**：每个人的提示词写法也会影响模型的实际表现。

### 提示词调优

1. **A/B 测试**：不同模型对提示词的响应不同，需要实测调整
2. **逐步细化**：从简单指令开始，遇到问题时再增加约束
3. **保持第一人称**：AI 视角写"我要做什么"，不是给用户的说明书

---

## 7. 扩展方向

### 联动其他功能

1. **积分系统**：完成任务记录积分，积分可兑换奖励  
   参考：[Android 背单词模块](https://github.com/Ephemera1117/android-vocabulary-learning)
2. **推送通知**：到期备忘主动推送提醒  
   参考：[Android AI 推送教程](https://github.com/Ephemera1117/android-ai-push-tutorial)
3. **数据可视化（可选）**：执行率趋势图、规则调整历史。如果你的规则比较简单、执行情况稳定，可能不需要做到这么细
4. **多设备同步**：通过 WebDAV 或云端数据库同步规则和备忘

### 进阶玩法（未实测）

以下是一些可能的扩展方向，笔者没有实操过，仅供参考：

1. **自动规则生成**：AI 根据执行数据自动提议新规则
2. **情境感知**：根据时间、地点、日程自动调整规则优先级
3. **长期趋势分析**：归档的备忘可用于回顾和总结

---

## 总结

这套代码实现的核心是：

1. **时间感知**：通过 Transformer 自动注入时间信息
2. **状态记录**：通过备忘录工具记录跨窗口的临时信息
3. **动态调整**：通过提示词引导 AI 根据执行数据调整规则
4. **工具驱动**：所有操作通过明确的工具接口，数据持久化到数据库

即使不是 Android 平台，这套逻辑也可以移植到 Web、iOS 或桌面应用，核心思路是通用的。

希望这份代码参考能帮助你实现自己的 AI 日常调教系统！
