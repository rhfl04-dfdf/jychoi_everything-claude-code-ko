# 비자 서류 번역기

이미지에서 비자 신청 서류를 자동으로 번역하여 전문적인 영문 PDF로 변환합니다.

## 기능

- 🔄 **자동 OCR**: 여러 OCR 방법 시도 (macOS Vision, EasyOCR, Tesseract)
- 📄 **이중 언어 PDF**: 원본 이미지 + 전문 영어 번역
- 🌍 **다국어 지원**: 중국어 및 기타 언어 지원
- 📋 **전문적인 형식**: 공식 비자 신청에 적합
- 🚀 **완전 자동화**: 수동 작업 불필요

## 지원 서류

- 은행 잔액 증명서 (存款证明)
- 재직 증명서 (在职证明)
- 퇴직 증명서 (退休证明)
- 소득 증명서 (收入证明)
- 부동산 증명서 (房产证明)
- 사업자 등록증 (营业执照)
- 신분증 및 여권

## 사용법

```bash
/visa-doc-translate <image-file>
```

### 예시

```bash
/visa-doc-translate RetirementCertificate.PNG
/visa-doc-translate BankStatement.HEIC
/visa-doc-translate EmploymentLetter.jpg
```

## 출력

`<filename>_Translated.pdf` 파일 생성:
- **1페이지**: 원본 문서 이미지 (중앙 정렬, A4 크기)
- **2페이지**: 전문 영어 번역

## 요구사항

### Python 라이브러리
```bash
pip install pillow reportlab
```

### OCR (다음 중 하나)

**macOS (권장)**:
```bash
pip install pyobjc-framework-Vision pyobjc-framework-Quartz
```

**크로스 플랫폼**:
```bash
pip install easyocr
```

**Tesseract**:
```bash
brew install tesseract tesseract-lang
pip install pytesseract
```

## 작동 방식

1. 필요한 경우 HEIC를 PNG로 변환
2. EXIF 회전 확인 및 적용
3. 사용 가능한 OCR 방법으로 텍스트 추출
4. 전문 영어로 번역
5. 이중 언어 PDF 생성

## 적합한 용도

- 🇦🇺 호주 비자 신청
- 🇺🇸 미국 비자 신청
- 🇨🇦 캐나다 비자 신청
- 🇬🇧 영국 비자 신청
- 🇪🇺 EU 비자 신청

## 라이선스

MIT
