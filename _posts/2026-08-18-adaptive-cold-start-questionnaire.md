---
title: "신규 사용자에게 무엇을 물어야 추천이 실제로 바뀔까"
date: 2026-08-18 17:00:00 +0900
categories: [AI]
tags: [recommendation, cold-start, onboarding]
---

온보딩 질문지를 짜다가 한참 뒤에 깨달았다. 나는 질문을 먼저 나열하고 있었다.

"학습 스타일은?", "경험 연수는?", "목표는?", "선호하는 형식은?". 종이 위에서는 그럴듯했다. 그런데 이 질문의 응답이 실제로 추천 결과를 바꾸는지는 한 번도 확인하지 않은 채였다. 질문을 먼저 만들고 로직은 나중에 맞추려던 셈이다.

방향이 반대였다.

## 카탈로그부터 열어봤어야 했다

맞는 순서는 사용자가 아니라 아이템 쪽에서 시작하는 것이다. 추천 시스템이 실제로 어떤 축으로 콘텐츠를 구분하는지 먼저 보면, 필요한 질문이 저절로 떠오른다. DB에서 아이템을 읽어 오는 자료구조를 열어봤다.

```python
@dataclass
class Template:
    project_id: str
    project_title: str
    tools: List[str] = field(default_factory=list)
    tool_names: List[str] = field(default_factory=list)
    categories: List[str] = field(default_factory=list)
    canvas_objects: List[str] = field(default_factory=list)
    activity_type: str = ""
    learning_goal: str = ""
    keywords: List[str] = field(default_factory=list)
    keywords_en: List[str] = field(default_factory=list)
    description: str = ""
    grade: Optional[int] = None
    semester: Optional[int] = None
    unit_code: str = ""
    school_level: str = ""  # 초등/중등/고등
```

콘텐츠 한 편이 갖는 필드가 이 정도다. 여기서 실제 필터·랭킹에 쓸 만한 것을 솎아내면 성격이 두 갈래로 갈린다.

한쪽은 `grade`, `school_level`, `activity_type`, `tools` 처럼 범주가 명확한 것들이다. `school_level` 이 "초등" 인 사용자에게 "고등" 콘텐츠를 넣을 이유가 없다. 이건 **필터** 다. 응답이 있으면 후보군 자체가 잘려나간다.

다른 쪽은 `learning_goal`, `keywords`, `description` 이다. 이건 사용자의 언어와 카탈로그의 언어가 얼마나 겹치느냐의 문제라서, 자르는 게 아니라 **정렬** 에 가깝다. 검색 쪽 문제다.

두 신호는 성격이 다르니 온보딩에서 얻는 방식도 달라야 했다. 필터로 쓸 값은 직접 물어서 못 박는 게 낫고, 정렬용 신호는 한 번에 다 받기도 어렵고 굳이 그럴 필요도 없다. 그런데 초기 질문지는 이 둘을 구분하지 않고 있었다.

## 응답 스키마가 이미 답을 하고 있었다

재밌었던 건 사용자 발화를 파싱하는 응답 모델 쪽이었다.

```python
class KeywordExtractionResponse(BaseResponse):
    user_input: str = Field(..., description="입력 텍스트")
    keywords: List[str] = Field(default_factory=list, description="추출된 키워드 목록")
    direct_keyword_hits: List[str] = Field(default_factory=list, description="직접 매칭된 키워드")
    is_educational: bool = Field(default=False, description="교육 목적 여부")
    education_context: Optional[str] = Field(default=None, description="교육 맥락 설명")
    subject_domain: Optional[str] = Field(default=None, description="과목 도메인")
    requested_school_level: Optional[str] = Field(default=None, description="요청된 학교급 (초등/중등/고등)")
    requested_grade: Optional[int] = Field(default=None, description="요청된 학년")
```

사용자가 자유 텍스트를 던지면 시스템은 거기서 `requested_school_level` 과 `requested_grade` 를 뽑아내려고 한다. 즉 구조적 신호를 온보딩에서 물어보지 않고 발화에서 끌어내겠다는 설계다. 첫 화면 UX 관점에서는 우아하다. 질문지가 짧아진다.

문제는 콜드스타트 시점이다. "수업에 쓸 자료 추천해줘" 같은 입력에서는 `requested_school_level` 도 `requested_grade` 도 `None` 이 되어 돌아온다. 반면 "중2 1학기 함수 단원 활동지" 같은 입력이면 필터 조건 대부분이 채워진다. 자유 텍스트 방식은 정보가 풍부할 때만 강하다. 빈약하면 그냥 `Optional[...] = None` 이 잔뜩 붙은 응답이 나올 뿐이다.

그러니까 여기서 판단이 갈렸다. 학교급·학년 같은 필터 값은 발화에서 뽑히길 기다리지 말고 온보딩에서 먼저 확정하는 게 낫다. 반대로 `keywords`, `learning_goal` 같은 정렬용 신호는 첫 발화가 들어올 때 뽑아도 늦지 않는다. 응답 스키마가 `Optional` 로 열어둔 지점과 아닌 지점이 이미 힌트를 주고 있었던 셈이다.

## 그래서 질문지의 판단 기준은 하나로 압축된다

이 응답을 알기 전과 후에 **보여줄 후보군이 실제로 달라지는가.**

달라지지 않는다면 그 질문은 온보딩 마찰만 늘리고 얻는 게 없다. "학습 스타일은?" 이 그런 종류였다. 카탈로그 스키마 어디에도 학습 스타일에 매핑되는 필드가 없다. 이 응답이 있어도 SQL 의 `WHERE` 절이 달라지지 않고, 랭킹에 넣을 대응 필드도 없다. 그러니 지금 물어봐야 할 이유가 없다. 나중에 학습 스타일에 반응하는 랭킹 항목을 실제로 만들면 그때 물으면 된다.

반대로 `school_level` 은 필터 그 자체다. 이 하나로 후보가 삼등분된다. 물어야 한다.

## 아직 안 본 곳

추천 응답에는 `relevance_score` 가 붙어 돌아온다.

```python
class RecommendedTemplateResponse(BaseModel):
    project_id: str
    project_title: str
    project_thumbnail: Optional[str] = Field(default=None)
    tools: List[str] = Field(default_factory=list)
    tool_names: List[str] = Field(default_factory=list)
    main_tool: str = Field(default="")
    main_tool_thumbnail: Optional[str] = Field(default=None)
    parent_tool: Optional[str] = Field(default=None)
    relevance_score: int = Field(..., description="연관도 점수 (0~100)")
    reason: str = Field(..., description="추천 이유")
```

0~100 의 정수다. 이 점수가 콜드스타트에서 어떻게 계산되는지는 아직 안 열어봤다. 사용자 이력이 없을 때 이 값이 무엇을 근거로 나오는지 — 카탈로그 평균 같은 상수인지, 구조적 필터를 통과한 뒤의 상대 점수인지, 아니면 LLM 이 `reason` 을 쓰면서 같이 만들어내는 값인지 — 에 따라 초기 추천의 성격이 달라진다. LLM 이 만들어내는 값이라면 재현성이 없고, 필터 통과율 기반이라면 사용자 응답 한두 개로 순위가 요동칠 수 있다.

솔직히 이 지점이 질문지 설계보다 더 결정적일 수 있다. 온보딩을 아무리 잘 짜도 `relevance_score` 계산이 콜드스타트에서 잡음이면 첫 화면은 무너진다. 아직 재보지 않았다. 다음 주에 여는 게 이쪽이다.