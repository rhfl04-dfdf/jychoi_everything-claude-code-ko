---
name: visa-doc-translate
description: 비자 신청 서류(이미지)를 영어로 번역하고 원본과 번역본이 포함된 이국어 PDF를 생성합니다.
---

비자 신청을 위한 비자 신청 서류 번역을 돕습니다.

## Instructions

사용자가 이미지 파일 경로를 제공하면, 확인을 요청하지 않고 다음 단계를 자동으로 실행합니다:

1. **이미지 변환 (Image Conversion)**: 파일이 HEIC인 경우, `sips -s format png <input> --out <output>`을 사용하여 PNG로 변환합니다.

2. **이미지 회전 (Image Rotation)**:
   - EXIF 방향 데이터를 확인합니다.
   - EXIF 데이터에 따라 이미지를 자동으로 회전합니다.
   - EXIF 방향이 6인 경우, 시계 반대 방향으로 90도 회전합니다.
   - 필요에 따라 추가 회전을 적용합니다 (문서가 거꾸로 보이면 180도 테스트).

3. **OCR 텍스트 추출 (OCR Text Extraction)**:
   - 여러 OCR 방법을 자동으로 시도합니다:
     - macOS Vision framework (macOS에서 선호됨)
     - EasyOCR (크로스 플랫폼, tesseract 불필요)
     - Tesseract OCR (사용 가능한 경우)
   - 문서에서 모든 텍스트 정보를 추출합니다.
   - 문서 유형(예금 증명서, 재직 증명서, 퇴직 증명서 등)을 식별합니다.

4. **번역 (Translation)**:
   - 모든 텍스트 내용을 전문적인 영어로 번역합니다.
   - 원본 문서의 구조와 형식을 유지합니다.
   - 비자 신청에 적합한 전문 용어를 사용합니다.
   - 고유 명사는 원어 그대로 유지하고 괄호 안에 영어를 병기합니다.
   - 중국 이름의 경우 병음(pinyin) 형식을 사용합니다 (예: WU Zhengye).
   - 모든 숫자, 날짜, 금액을 정확하게 보존합니다.

5. **PDF 생성 (PDF Generation)**:
   - PIL 및 reportlab 라이브러리를 사용하는 Python 스크립트를 작성합니다.
   - Page 1: 회전된 원본 이미지를 A4 페이지에 맞게 중앙에 배치하고 크기를 조정하여 표시합니다.
   - Page 2: 적절한 서식과 함께 영어 번역본을 표시합니다:
     - 제목은 중앙 정렬 및 굵게 표시
     - 내용은 적절한 간격으로 왼쪽 정렬
     - 공식 문서에 적합한 전문적인 레이아웃
   - 하단에 노트를 추가합니다: "This is a certified English translation of the original document"
   - 스크립트를 실행하여 PDF를 생성합니다.

6. **출력 (Output)**: 동일한 디렉토리에 `<original_filename>_Translated.pdf`라는 이름의 PDF 파일을 생성합니다.

## Supported Documents

- 은행 예금 증명서 (存款证明)
- 소득 증명서 (收入证明)
- 재직 증명서 (在职证明)
- 퇴직 증명서 (退休证明)
- 부동산 증명서 (房产证明)
- 사업자 등록증 (营业执照)
- 신분증 및 여권
- 기타 공식 문서

## Technical Implementation

### OCR Methods (tried in order)

1. **macOS Vision Framework** (macOS 전용):
   ```python
   import Vision
   from Foundation import NSURL
   ```

2. **EasyOCR** (크로스 플랫폼):
   ```bash
   pip install easyocr
   ```

3. **Tesseract OCR** (사용 가능한 경우):
   ```bash
   brew install tesseract tesseract-lang
   pip install pytesseract
   ```

### Required Python Libraries

```bash
pip install pillow reportlab
```

macOS Vision framework의 경우:
```bash
pip install pyobjc-framework-Vision pyobjc-framework-Quartz
```

## Important Guidelines

- 각 단계마다 사용자 확인을 요청하지 마십시오.
- 최적의 회전 각도를 자동으로 결정하십시오.
- 하나가 실패하면 여러 OCR 방법을 시도하십시오.
- 모든 숫자, 날짜, 금액이 정확하게 번역되었는지 확인하십시오.
- 깔끔하고 전문적인 서식을 사용하십시오.
- 전체 프로세스를 완료하고 최종 PDF 위치를 보고하십시오.

## Example Usage

```bash
/visa-doc-translate RetirementCertificate.PNG
/visa-doc-translate BankStatement.HEIC
/visa-doc-translate EmploymentLetter.jpg
```

## Output Example

이 skill은 다음을 수행합니다:
1. 사용 가능한 OCR 방법을 사용하여 텍스트 추출
2. 전문적인 영어로 번역
3. 다음이 포함된 `<filename>_Translated.pdf` 생성:
   - Page 1: 원본 문서 이미지
   - Page 2: 전문적인 영어 번역본

호주, 미국, 캐나다, 영국 및 번역된 서류가 필요한 기타 국가의 비자 신청에 적합합니다.
