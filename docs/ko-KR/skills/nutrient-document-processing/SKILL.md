---
name: nutrient-document-processing
description: Nutrient DWS API를 사용하여 문서를 처리, 변환, OCR, 추출, 교정(redact), 서명 및 작성합니다. PDF, DOCX, XLSX, PPTX, HTML 및 이미지와 호환됩니다.
origin: ECC
---

# Nutrient Document Processing

[Nutrient DWS Processor API](https://www.nutrient.io/api/)를 사용하여 문서를 처리하세요. 형식을 변환하고, 텍스트와 테이블을 추출하며, 스캔된 문서를 OCR 처리하고, PII(개인식별정보)를 교정(redact)하며, 워터마크 추가, 디지털 서명 및 PDF 폼 작성이 가능합니다.

## 설정

**[nutrient.io](https://dashboard.nutrient.io/sign_up/?product=processor)**에서 무료 API 키를 받으세요.

```bash
export NUTRIENT_API_KEY="pdf_live_..."
```

모든 요청은 `instructions` JSON 필드가 포함된 multipart POST 형식으로 `https://api.nutrient.io/build`에 전송됩니다.

## 작업

### 문서 변환

```bash
# DOCX를 PDF로
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.docx=@document.docx" \
  -F 'instructions={"parts":[{"file":"document.docx"}]}' \
  -o output.pdf

# PDF를 DOCX로
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"output":{"type":"docx"}}' \
  -o output.docx

# HTML을 PDF로
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "index.html=@index.html" \
  -F 'instructions={"parts":[{"html":"index.html"}]}' \
  -o output.pdf
```

지원되는 입력 형식: PDF, DOCX, XLSX, PPTX, DOC, XLS, PPT, PPS, PPSX, ODT, RTF, HTML, JPG, PNG, TIFF, HEIC, GIF, WebP, SVG, TGA, EPS.

### 텍스트 및 데이터 추출

```bash
# 일반 텍스트 추출
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"output":{"type":"text"}}' \
  -o output.txt

# 테이블을 Excel로 추출
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"output":{"type":"xlsx"}}' \
  -o tables.xlsx
```

### 스캔된 문서 OCR

```bash
# OCR을 통해 검색 가능한 PDF로 변환 (100개 이상의 언어 지원)
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "scanned.pdf=@scanned.pdf" \
  -F 'instructions={"parts":[{"file":"scanned.pdf"}],"actions":[{"type":"ocr","language":"english"}]}' \
  -o searchable.pdf
```

언어: ISO 639-2 코드(예: `eng`, `deu`, `fra`, `spa`, `jpn`, `kor`, `chi_sim`, `chi_tra`, `ara`, `hin`, `rus`)를 통해 100개 이상의 언어를 지원합니다. `english` 또는 `german`과 같은 전체 언어 이름도 사용할 수 있습니다. 지원되는 모든 코드는 [전체 OCR 언어 표](https://www.nutrient.io/guides/document-engine/ocr/language-support/)를 참조하세요.

### 민감 정보 교정(Redact)

```bash
# 패턴 기반 (SSN, 이메일)
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"redaction","strategy":"preset","strategyOptions":{"preset":"social-security-number"}},{"type":"redaction","strategy":"preset","strategyOptions":{"preset":"email-address"}}]}' \
  -o redacted.pdf

# 정규식(Regex) 기반
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"redaction","strategy":"regex","strategyOptions":{"regex":"\\b[A-Z]{2}\\d{6}\\b"}}]}' \
  -o redacted.pdf
```

프리셋: `social-security-number`, `email-address`, `credit-card-number`, `international-phone-number`, `north-american-phone-number`, `date`, `time`, `url`, `ipv4`, `ipv6`, `mac-address`, `us-zip-code`, `vin`.

### 워터마크 추가

```bash
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"watermark","text":"CONFIDENTIAL","fontSize":72,"opacity":0.3,"rotation":-45}]}' \
  -o watermarked.pdf
```

### 디지털 서명

```bash
# 자체 서명된 CMS 서명
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"sign","signatureType":"cms"}]}' \
  -o signed.pdf
```

### PDF 폼 작성

```bash
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "form.pdf=@form.pdf" \
  -F 'instructions={"parts":[{"file":"form.pdf"}],"actions":[{"type":"fillForm","formFields":{"name":"Jane Smith","email":"jane@example.com","date":"2026-02-06"}}]}' \
  -o filled.pdf
```

## MCP Server (대안)

네이티브 도구 통합을 위해 curl 대신 MCP server를 사용하세요:

```json
{
  "mcpServers": {
    "nutrient-dws": {
      "command": "npx",
      "args": ["-y", "@nutrient-sdk/dws-mcp-server"],
      "env": {
        "NUTRIENT_DWS_API_KEY": "YOUR_API_KEY",
        "SANDBOX_PATH": "/path/to/working/directory"
      }
    }
  }
}
```

## 사용 사례

- 문서 형식 간 변환 (PDF, DOCX, XLSX, PPTX, HTML, 이미지)
- PDF에서 텍스트, 테이블 또는 키-값 쌍 추출
- 스캔된 문서 또는 이미지에 대한 OCR
- 문서 공유 전 PII(개인식별정보) 교정
- 초안 또는 기밀 문서에 워터마크 추가
- 계약서 또는 합의서에 디지털 서명
- 프로그래밍 방식으로 PDF 폼 작성

## 링크

- [API Playground](https://dashboard.nutrient.io/processor-api/playground/)
- [전체 API 문서](https://www.nutrient.io/guides/dws-processor/)
- [npm MCP Server](https://www.npmjs.com/package/@nutrient-sdk/dws-mcp-server)
