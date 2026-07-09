# windforce-docs

[windforce](https://imprun.dev) 콘솔·플랫폼 공개 문서 사이트 소스 — [docs.imprun.dev](https://docs.imprun.dev).

MkDocs Material로 빌드하고 GitHub Actions → GitHub Pages로 배포한다.

```bash
pip install -r requirements-docs.txt
mkdocs serve          # 로컬 미리보기 http://127.0.0.1:8000
mkdocs build --strict # 링크·내비 검증
```
