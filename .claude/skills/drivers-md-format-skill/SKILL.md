---
name: drivers-md-format-skill
description: 'Write, continue, or reorganize Chinese Linux driver-study Markdown as a coherent professional textbook chapter. Use this skill whenever the user supplies driver-learning notes, rough anchors, APIs, code, chapter topics, or asks to continue a Linux driver, device-model, DeviceTree, platform-driver, sysfs, kobject, character-device, timer, poll, or similar tutorial. Transform every substantive user-provided point into the document, connect fragmented anchors with accurate explanatory material, and use numbered headings, API prototypes, examples, version differences, and cleanup guidance as appropriate.'
argument-hint: 'Topic or chapter to write in the established driver-note format'
user-invocable: true
disable-model-invocation: false
---

# Driver Markdown Format Skill

## Purpose

Use this skill to produce or extend driver-study Markdown in the same format as the earlier Drivers.md document.

The output should read like a Chinese textbook or training handout:
- Structured by numbered heading levels
- Focused on kernel driver concepts and APIs
- Mixing explanation, code prototypes, parameter descriptions, examples, and practical notes
- Keeping style consistent across chapters
- Written as an author creating a professional document, not as an assistant replying to a question
- Using an objective third-person or impersonal technical perspective wherever natural

## When to Use

Use this skill when the user asks to:
- Write a new driver chapter in Chinese
- Continue an existing Drivers.md-style document
- Reformat rough notes into textbook-style Markdown
- Add sections such as timer, poll, lseek, chrdev, private_data, registration flow, cleanup flow
- Normalize heading levels like `1`, `1.1`, `1.1.1`
- Explain kernel APIs with definitions, parameters, return values, example code, and version differences

## Target Style

The document should follow these style rules:
- Main chapter uses `#` and numbered titles such as `# 1 字符设备驱动`
- Second level uses `##` and numbered titles such as `## 1.1 字符设备注册`
- Third level uses `###` and numbered titles such as `### 1.1.1 申请设备号`
- If a fourth level is needed, use `####` with continued numbering such as `#### 4.5.1.1 旧版本`
- Titles should be concise and topic-first, not conversational
- Content should be in Chinese, technically accurate, and suitable for teaching
- Write declarative textbook prose. Do not use dialogue-like wording such as “你可以”“你需要”“下面回答” or “这个问题”; instead state mechanisms, conditions, and conclusions directly.
- Treat the user's supplied material as chapter requirements. Every substantive point, code example, API name, scenario, or requested relationship must appear in the resulting document in an appropriate location.
- The user may provide only a sequence of anchors. Expand those anchors into a complete, logically connected chapter; supply the transitions, prerequisites, consequences, and necessary background that make the chapter independently readable.
- When an anchor contains a technical inaccuracy, preserve its intended teaching topic but write the accurate formulation naturally. Do not add editorial statements such as “已更正”“用户说法错误” or describe the correction process in the document.
- Prefer short paragraphs and explicit bullet points over dense prose
- Use fenced C code blocks for API prototypes and examples
- Use tables only when they clearly improve understanding, such as macro parameter mapping

## Standard Chapter Pattern

For each major topic, organize content using this progression when applicable:

1. Concept introduction
2. Core API or mechanism
3. Function prototype or macro definition
4. Parameter explanation
5. Return value or behavior explanation
6. Calling sequence or usage timing
7. Minimal example code
8. Version differences or old/new API comparison
9. Notes, pitfalls, or cleanup order

Not every section needs all nine parts, but the order should remain stable when they apply.

## Heading Pattern

Use the following numbering pattern consistently:

```md
# 1 Chapter Title
## 1.1 Section Title
### 1.1.1 Subsection Title
### 1.1.2 Subsection Title
## 1.2 Section Title
### 1.2.1 Subsection Title
```

For API evolution or split cases, use a deeper level only when needed:

```md
### 4.5.1 DEFINE_TIMER
#### 4.5.1.1 旧版本
#### 4.5.1.2 新版本（Linux 4.15 及以上）
```

## Content Templates

### Concept Template

```md
### 1.1.1 某概念

某机制的定义、作用和使用场景。

核心作用：
- 作用 1
- 作用 2
- 作用 3
```

### API Template

```md
### 1.1.2 某函数

```c
int example_function(type arg1, type arg2);
```

参数说明：
- `arg1`：含义
- `arg2`：含义

返回值说明：
- `0`：成功
- `< 0`：失败，返回负错误码
```

### Example Template

```md
示例：

```c
static int __init demo_init(void)
{
    return 0;
}
```
```

### Cleanup or Rollback Template

```md
推荐回滚顺序：

1. 先释放最后创建的资源
2. 再释放更早创建的资源
3. 保证销毁顺序与创建顺序相反

核心原则：
- 谁先创建，谁后销毁
- 出错路径和退出路径都要完整
```

### Version Comparison Template

```md
旧版本特点：
- 接口形式
- 回调原型
- 常见限制

新版本特点：
- 新接口形式
- 新回调原型
- 推荐写法
```

## Driver-Specific Writing Rules

When documenting driver code, prefer these content blocks:
- Registration order for character devices
- `open`, `read`, `write`, `llseek`, `poll`, timer callback responsibilities
- Meaning of `file->private_data`
- Relationship between user-space behavior and kernel-side callbacks
- Resource allocation and rollback order
- Old and new kernel API differences when relevant

When explaining code behavior:
- Distinguish user-space concepts from kernel-space concepts
- State whether an API changes position, data, state, or registration only
- Explain why a bug happens, not just how to patch it
- For string and buffer examples, distinguish raw bytes from C strings ending in `\0`

## Procedure

1. Extract every substantive anchor from the user's current request: required concepts, APIs, structures, code, scenarios, source paths, relationships, version constraints, and requested examples.
2. Identify the chapter topic and place it in the existing numbering hierarchy.
3. Order the anchors into a teaching path. Begin with prerequisite concepts, then mechanisms and APIs, then usage flow, examples, consequences, and cautions.
4. Decide the correct heading depth before writing body text.
5. Start with a concise concept paragraph that establishes why the topic exists and what problem it solves.
6. Add the relevant prototype, macro, or structure definition in a fenced C block.
7. Explain parameters or fields using bullets.
8. Add return values or behavioral notes if the API has them.
9. Provide a minimal example matching the current kernel style.
10. If kernel versions differ, split old and new usage into separate subsections.
11. Add notes about common mistakes, cleanup order, call timing, and concurrency when the topic needs them.
12. Verify that every extracted anchor is represented, then check numbering continuity, terminology consistency, and logical transitions across the chapter.

## Decision Points

Choose structure based on topic type:

- If the topic is a registration or teardown flow: emphasize order, rollback, and paired APIs.
- If the topic is a single API: emphasize prototype, parameters, return value, and example.
- If the topic changed across kernel versions: split into old/new subsections.
- If the topic is a behavioral mechanism like `lseek`, `poll`, or timers: explain both kernel implementation and user-space effect.
- If the topic is easy to misuse: add a dedicated "注意事项" block.

## Quality Checks

Before finishing, verify:
- Every substantive item in the user's current message is represented in the document; none is silently dropped merely because it was supplied as a short anchor.
- The chapter reads as a standalone professional teaching document rather than a conversational answer.
- The prose uses an objective third-person or impersonal perspective and avoids direct second-person instruction unless a command example requires it.
- Concepts appear before the APIs and examples that depend on them, and each section has a clear transition to the next.
- Any inaccurate user-provided detail has been replaced with the technically correct explanation without discussing the correction in the document.
- Numbered headings are continuous and at the correct depth.
- Similar sections use parallel structure.
- All code blocks are fenced and language-tagged as `c` when appropriate.
- Parameter and return-value explanations match the shown prototype.
- Old/new API descriptions do not contradict each other.
- Terminology is stable across the document.
- Example code matches the claimed kernel version.
- Cleanup and error rollback steps are in reverse creation order.
- String, buffer, jiffies, timer, and pointer explanations are technically accurate.

## Output Expectations

The final Markdown should be:
- Readable as teaching material
- Suitable for continuing an existing Drivers.md-style handout
- Consistent in numbering, tone, and formatting
- Detailed enough for a learner to follow without guessing missing steps

## Example Prompts

- Write a new section on `llseek` in the same Drivers.md style.
- Continue chapter 4 and add a subsection for timer deletion and synchronization.
- Reformat these rough notes into the numbered driver textbook format.
- Add an old/new kernel API comparison section for `DEFINE_TIMER`.
