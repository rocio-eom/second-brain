# second-brain

개인 기술 지식 베이스. Obsidian vault.

PARA + Zettelkasten Permanent Notes + MOC 하이브리드 방법론을 따른다.

---

## 방법론

| 요소 | 출처 | 역할 |
|---|---|---|
| PARA | Tiago Forte | lifecycle 폴더 구조 (Projects / Areas / Resources / Archive) |
| Permanent Notes | Niklas Luhmann (Zettelkasten) | atomic·evergreen 개념 노트 |
| MOC (Map of Content) | Nick Milo | 토픽별 인덱스. 링크만 보유, 콘텐츠 없음 |

캡처는 빠르게 (Inbox), 가공은 천천히 (Fleeting → Permanent), 탐색은 두 갈래로 (도메인 MOC / lifecycle 폴더).

---

## 폴더 구조

| Folder | Role |
|---|---|
| `00_Inbox/` | 즉시 캡처 (분류 없이, 30초 룰). `daily/` 하위에 daily note 보관 |
| `01_Fleeting/` | 확장 노트. 48h 내 Permanent 승격 또는 삭제 |
| `02_Literature/` | 외부 자료 정리 (책·논문·영상·코스). **optional** |
| `03_Permanent/` | atomic 개념 노트. `{domain}/{sub-domain}/` 2-depth |
| `04_MOC/` | 토픽 인덱스. `{domain}/` 1-depth. 링크만 |
| `05_Projects/` | 시한 있는 학습/구현 작업. `_active/` `_archive/` |
| `06_Areas/` | 지속 관리 영역 (Career, Reading-List 등) |
| `07_Decisions/` | ADR + 회고. `Architecture/` `Tech-Selection/` |
| `08_Templates/` | 노트 타입별 템플릿 + `tag-taxonomy.md` |
| `09_Archive/` | 종료/비활성 자료 (동결, 삭제 아님) |

### Permanent 도메인

```
03_Permanent/AI-ML-LLM/{RAG, Embedding, Vector-Search, LLM-Serving, AWS-Bedrock, Chunking, LLM-Security}/
03_Permanent/Backend/{Distributed-Systems, Data-Pipeline, AWS}/
03_Permanent/Frontend/         (flat)
03_Permanent/CS-Fundamentals/  (flat)
```

서브폴더는 동일 도메인 내 노트 ≥10건 시 분할.

---

## 노트 lifecycle

```
Inbox → Fleeting → (Literature →) Permanent → MOC 갱신
```

- Fleeting → Permanent: 새 파일을 `03_Permanent/{domain}/{sub}/` 아래 생성하고, 원본 Fleeting 삭제 (in-place replacement 아님).
- Permanent 작성 후: `moc` 필드 채우고 해당 MOC 파일에도 링크 라인 추가.
- Literature 단계는 optional. 짧은 외부 자료는 Fleeting의 `## 참고 자료` 섹션으로 충분.
- Decision: 작성 → 2–4주 후 `## Retrospective` 섹션 추가.

---

## 파일 명명 규칙

| Type | Pattern |
|---|---|
| Fleeting | `fl-YYYY-MM-DD-{slug}.md` |
| Literature | `lit-{source}-{topic}.md` |
| Permanent | `{concept-slug}.md` (날짜 없음) |
| MOC | `moc-{topic-slug}.md` |
| Decision | `dec-YYYY-MM-DD-{slug}.md` |
| Project | `prj-{slug}.md` |
| Daily | `00_Inbox/daily/YYYY-MM-DD.md` |

소문자 + 하이픈. 인식 가능한 약어는 대문자 유지 (예: `RAG-pipeline.md`).
