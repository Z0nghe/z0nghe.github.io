---
layout: post
title:  "AI 钢琴基础教师"
date:   2026-06-28 20:30:00 +0800
categories: ai piano agent music
abstract: "基于本地钢琴教师 Agent 项目，分析一个 AI 钢琴基础教师的模块设计和练习反馈流程"
---

# AI 钢琴基础教师

## 一、项目目标

`piano_teacher` 是一个本地钢琴练习助手。它不是简单地把问题交给大模型回答，而是把谱面导入、`MusicXML` 标准化、MIDI 练习记录、音符级对齐、结构化分析、长期练习画像和教学反馈串成一个完整的练琴闭环。

从使用者的角度看，可以直接用自然语言说：

```text
准备 minimal-piano-fixture
mock 听我练习 minimal-piano-fixture 前24个音 rough 模式
查看 minimal-piano-fixture 的学习进度和计划
讲一下节奏和拍子
```

从工程角度看，这个系统更像一个“确定性分析服务 + 教学表达 Agent”的组合。谱面解析、MIDI 对齐和评分由本地代码完成，大模型或子 Agent 主要负责把分析结果转化成适合初学者理解的练习建议。

## 二、整体模块划分

项目的核心代码放在 `app/` 目录下：

```text
app/
  agents/
    conductor_agent.py
    piano_teacher_agent.py
    score_import_review_agent.py
  services/
    score_normalizer_service.py
    midi_parsing_service.py
    alignment_service.py
    analysis_service.py
    feedback_service.py
    profile_service.py
  tools/
    musicxml_reader.py
    score_aligner.py
    midi_parser.py
  workflows/
    start_practice_workflow.py
    finish_practice_workflow.py
    listen_mock_practice_workflow.py
```

其中几个关键职责如下：

* `ConductorAgent`：自然语言主控，判断用户想做什么，并调用对应 workflow。
* `MusicXmlReader`：读取 `MusicXML` 或 `.mxl`，转换成统一的 `normalized_score.json`。
* `ScoreAligner`：将谱面音符和实际演奏 MIDI 事件逐个对齐。
* `AnalysisService`：根据对齐结果计算错音、漏音、节奏偏差和练习分数。
* `FeedbackService`：组合曲目信息、练习分析、长期画像和乐理知识，生成教学反馈。
* `ProfileService`：维护每首曲子的长期学习进度。

这样的划分有一个明显好处：大模型不直接决定“你弹错了哪个音”，而是读取本地程序已经计算出的结构化结果。这样反馈更稳定，也更容易测试。

## 三、自然语言主控

`ConductorAgent` 是系统入口。它接收用户的一句话，然后根据关键词和曲目识别结果选择具体命令。

```python
def handle_natural_language(self, text: str) -> dict:
    lowered = text.lower()
    piece_id = self._detect_piece_id(text)

    if self._has_any(lowered, ["help", "帮助", "能做什么", "怎么用"]):
        return self._help_response(piece_id)

    if self._is_theory_question(lowered):
        return self.handle_command("explain_theory", query=text)

    if self._has_any(lowered, ["prepare", "准备", "标准化", "normalize"]):
        if not piece_id:
            raise ValueError("请指定要准备的 piece_id 或曲名。")
        return self.handle_command("prepare_piece", piece_id=piece_id)

    if self._has_any(lowered, ["mock", "模拟", "听我", "listen"]):
        if not piece_id:
            raise ValueError("请指定要练习的 piece_id 或曲名。")
        return self.handle_command(
            "listen_mock_practice",
            piece_id=piece_id,
            mock_mode=self._detect_mock_mode(lowered),
            max_notes=self._detect_max_notes(lowered),
            user_command=text,
        )
```

这个实现没有做复杂的意图分类模型，而是用关键词完成第一版路由。对一个本地工具来说，这种方式很实用：可解释、容易调试，用户说法固定以后命中率也足够高。

真正执行动作的是 `handle_command()`：

```python
def handle_command(self, command: str, **kwargs) -> dict:
    if command == "refresh_library":
        return RefreshLibraryWorkflow(self.score_root).run()
    if command == "prepare_piece":
        refreshed = RefreshLibraryWorkflow(self.score_root).run()
        piece_id = kwargs.get("piece_id")
        normalized = NormalizeScoreWorkflow(self.score_root).run(piece_id)
        return {"status": "prepared", "library": refreshed, "normalized": normalized}
    if command == "listen_mock_practice":
        listened = ListenMockPracticeWorkflow(self.score_root, self.session_root).run(
            piece_id=kwargs["piece_id"],
            mock_mode=kwargs.get("mock_mode") or "rough",
            max_notes=int(kwargs.get("max_notes") or 96),
            user_command=kwargs.get("user_command"),
        )
        finished = FinishPracticeWorkflow(
            self.score_root,
            self.session_root,
            self.profile_root,
        ).run(listened["session"]["session_id"], listened["midi_log_path"])
        return {"status": "completed", "listening": listened, **finished}
```

可以看到，“听我练习”并不是一句聊天回复，而是连续跑了两个流程：

1. 创建练习会话并生成或接收 MIDI 演奏数据。
2. 结束练习，解析 MIDI，对齐谱面，生成分析和反馈。

## 四、谱面标准化

钢琴练习分析的第一步是把曲谱转换成机器容易处理的结构。项目中使用 `ScoreNormalizerService` 封装这个动作：

```python
class ScoreNormalizerService:
    def __init__(self) -> None:
        self.reader = MusicXmlReader()

    def normalize(self, piece_id: str, musicxml_path: str | Path, output_path: str | Path) -> dict:
        normalized = self.reader.normalize(piece_id, musicxml_path)
        write_json(output_path, normalized)
        return normalized
```

真正解析 `MusicXML` 的逻辑在 `MusicXmlReader` 中。它会读取标题、作曲者、拍号、调号、音符、时值、声部、左右手信息等内容：

```python
for measure_index, measure in enumerate(self._findall(root, ".//m:measure", ".//measure"), start=1):
    measure_number = int(measure.attrib.get("number", measure_index)) \
        if measure.attrib.get("number", str(measure_index)).isdigit() else measure_index
    onset_by_voice: dict[str, int] = {}
    notes = []
    for note_index, note in enumerate(self._findall(note := measure, "m:note", "note"), start=1):
        voice = self._text(note, "m:voice", "voice") or "1"
        duration = int(self._text(note, "m:duration", "duration") or "0")
        pitch_midi, pitch_name = self._pitch(note)
        staff = self._text(note, "m:staff", "staff")
        notes.append(
            {
                "note_id": f"{piece_id}-m{measure_number}-n{note_index}",
                "pitch_midi": pitch_midi,
                "pitch_name": pitch_name,
                "voice": int(voice) if voice.isdigit() else None,
                "staff": staff,
                "hand": self._hand_from_staff(staff),
                "is_rest": self._find(note, "m:rest", "rest") is not None,
                "onset_division": onset_by_voice.get(voice, 0),
                "duration_division": duration,
            }
        )
```

这里最关键的是 `pitch_midi` 和 `onset_division`。前者把音高变成 MIDI 数字，后者表示音符在小节内的起始位置。这样后续对齐时就不需要再理解 XML 树，只需要比较标准化 JSON。

左右手判断也做得很朴素：

```python
def _hand_from_staff(self, staff: str | None) -> str:
    if staff == "1":
        return "right"
    if staff == "2":
        return "left"
    return "unknown"
```

在钢琴谱里，高音谱表通常是右手，低音谱表通常是左手。这个规则不一定覆盖所有复杂谱面，但对基础练习已经够用。

## 五、MIDI 与谱面对齐

有了标准谱面和实际演奏以后，系统需要回答一个问题：用户每一个音弹得对不对，早了还是晚了，有没有漏音或多弹。

`ScoreAligner` 的第一版实现采用顺序对齐：

```python
for index, score_note in enumerate(score_notes):
    event = events[index] if index < len(events) else None
    if event is None:
        note_matches.append(self._match(score_note, None, "missed"))
        continue

    expected_pitch = score_note.get("pitch_midi")
    performed_pitch = event.get("pitch_midi")
    timing_offset = self._expected_start_ms(score, score_note) - int(event.get("start_ms", 0))

    if expected_pitch != performed_pitch:
        match_type = "pitch_error"
    elif abs(timing_offset) > timing_tolerance_ms:
        match_type = "timing_error"
    else:
        match_type = "matched"
    note_matches.append(self._match(score_note, event, match_type, timing_offset))
```

这段代码做了三件事：

1. 谱面上有音，但演奏里没有对应事件，标记为 `missed`。
2. 音高不同，标记为 `pitch_error`。
3. 音高相同但起音偏差超过阈值，标记为 `timing_error`。

默认节奏阈值是 180ms：

```python
timing_tolerance_ms = int(options.get("timing_tolerance_ms", 180))
```

第一版还假设速度为 120 bpm：

```python
def _expected_start_ms(self, score: dict, note: dict) -> int:
    divisions = max(int(score.get("division_unit") or 1), 1)
    # First version assumes 120 bpm, quarter note = 500 ms.
    return int(((note["measure_number"] - 1) * 4 + note.get("onset_division", 0) / divisions) * 500)
```

这个实现很适合作为基础教师的 MVP：先解决“音高是否正确、节奏是否明显偏离、小节问题在哪里”。如果后续要支持更自由的演奏速度，就可以在这里引入动态时间规整、节拍估计或更复杂的 MIDI-score alignment 算法。

## 六、练习分析

对齐结果只是一堆 `note_matches`，还不能直接给学生看。`AnalysisService` 会把它们汇总成准确率、评分、问题小节和下一步建议。

```python
matched = [item for item in matches if item.get("match_type") == "matched"]
missed = [item for item in matches if item.get("match_type") == "missed"]
extra = [item for item in matches if item.get("match_type") == "extra"]
pitch_errors = [item for item in matches if item.get("match_type") == "pitch_error"]
timing_errors = [item for item in matches if item.get("match_type") == "timing_error"]

denominator = max(expected_note_count, 1)
pitch_accuracy = max(0.0, 1.0 - (len(missed) + len(pitch_errors)) / denominator)
rhythm_accuracy = max(0.0, 1.0 - len(timing_errors) / denominator)
overall_score = round((pitch_accuracy * 0.65 + rhythm_accuracy * 0.35) * 100, 1)
```

这里的评分权重是：音高准确率 65%，节奏准确率 35%。对基础钢琴学习来说，这个权重比较合理。初学阶段最先要保证的是音弹对，再逐步处理节奏稳定、速度和音乐性。

接着，系统会统计问题集中在哪些小节：

```python
by_measure: dict[int, Counter] = defaultdict(Counter)
by_measure_hand: dict[int, Counter] = defaultdict(Counter)

for item in matches:
    match_type = item.get("match_type")
    if match_type == "matched":
        continue
    measure_number = item.get("measure_number")
    if isinstance(measure_number, int):
        by_measure[measure_number][match_type] += 1
        by_measure_hand[measure_number][item.get("hand", "unknown")] += 1
```

最后生成练习建议：

```python
def _next_steps(self, measures: list[int], issue_tags: list[str]) -> list[str]:
    if not measures:
        return ["保持当前速度完整弹奏一遍，并记录是否仍能稳定。"]
    measure_text = "、".join(str(item) for item in measures)
    if "pitch_error" in issue_tags or "missed" in issue_tags:
        return [f"先分手慢练第 {measure_text} 小节，连续三遍音高无错后再合手。"]
    if "timing_error" in issue_tags:
        return [f"用节拍器慢练第 {measure_text} 小节，确认每个起音都贴住拍点。"]
    return [f"循环第 {measure_text} 小节，先稳定准确率，再考虑提速。"]
```

这段逻辑已经很像一个基础钢琴老师的课堂语言：先定位小节，再判断问题类型，然后给一个可执行的练习动作。

## 七、教学反馈生成

`FeedbackService` 把曲目信息、练习会话、分析结果、长期学习画像和乐理上下文组织起来，交给 `PianoTeacherAgent`：

```python
class FeedbackService:
    def generate(self, piece: dict, session: dict, analysis: dict, profile: dict | None) -> dict:
        return self.teacher.generate_feedback(
            {
                "piece": {
                    "piece_id": piece.get("piece_id"),
                    "title": piece.get("title"),
                    "composer": piece.get("composer"),
                    "current_stage": piece.get("current_stage"),
                },
                "session": {
                    "session_id": session.get("session_id"),
                    "practice_mode": session.get("practice_mode"),
                    "tempo_target_bpm": session.get("tempo_target_bpm"),
                },
                "analysis": analysis,
                "profile": profile,
                "theory_context": self.theory.for_practice_context(analysis),
            }
        )
```

`PianoTeacherAgent` 有两种模式。如果配置了 Hermes 子 Agent，可以调用外部模型生成反馈；否则使用确定性反馈：

```python
def generate_feedback(self, payload: dict) -> dict:
    if self.review_command and os.getenv("PIANO_TEACHER_USE_HERMES_SUBAGENTS") == "1":
        model_feedback = self._generate_with_command(payload)
        if model_feedback:
            return model_feedback
    return self._generate_deterministic_feedback(payload)
```

确定性反馈会生成固定结构：

```python
return {
    "summary": summary,
    "key_issues": key_issues,
    "practice_plan": practice_plan,
    "progress_note": self._progress_note(profile),
    "next_focus": (analysis.get("recommended_next_steps") or [None])[0],
    "confidence": "medium" if warnings else "high",
}
```

这种设计比“直接让大模型看 MIDI 数据并自由发挥”更稳。因为输出至少包含 `summary`、`key_issues`、`practice_plan`、`next_focus` 这些固定字段，前端、CLI 或后续自动化都可以继续消费。

## 八、完整练习闭环

结束练习时，`FinishPracticeWorkflow` 会把所有模块串起来：

```python
performance = self.midi_parser.parse(session["midi_log_path"])
alignment = self.alignment.align(session["normalized_score_path"], performance)
normalized_score = read_json(session["normalized_score_path"])
expected_count = sum(
    1
    for measure in normalized_score["measures"]
    for note in measure["notes"]
    if not note["is_rest"]
)
performed_count = len([event for event in performance["events"] if event.get("type") == "note"])
analysis = self.analysis.analyze(
    session["session_id"],
    session["piece_id"],
    alignment,
    expected_count,
    performed_count,
)
profile = self.profile_service.update_from_analysis(session["piece_id"], analysis)
feedback = self.feedback.generate(piece, session, analysis, profile)
```

整个流程可以概括为：

```text
MusicXML/MXL/PDF
    -> normalized_score.json
    -> start practice session
    -> MIDI events
    -> score alignment
    -> practice analysis
    -> profile update
    -> teacher feedback
```

这里最值得注意的是 `profile`。一次练习反馈只能说明当下情况，而长期画像可以记录最近分数、当前目标、下一步计划和老师关注点。对于基础学习来说，“连续几次都卡在第 8 小节左手节奏”比单次错音更有教学价值。

## 九、总结

这个 AI 钢琴基础教师项目的重点不在于让大模型“假装懂音乐”，而是用工程方式把音乐学习拆成可验证的几个步骤：

1. 谱面先标准化，确保教师和学生讨论的是同一份结构化曲谱。
2. MIDI 与谱面对齐，用程序找出错音、漏音和节奏偏差。
3. 分析服务把底层错误汇总成小节、左右手、准确率和练习建议。
4. 教学 Agent 负责解释、鼓励和组织下一步计划。
5. 长期画像记录每首曲子的学习过程，让反馈从“这次弹得怎么样”变成“最近应该怎么练”。

第一版实现还有一些明显可以增强的地方，例如更准确的速度估计、更智能的多声部对齐、真实硬件 MIDI 录制、PDF 转谱后的人工复核流程等。但从代码结构上看，它已经把一个 AI 教师最重要的边界划清楚了：程序负责事实判断，Agent 负责教学表达。
