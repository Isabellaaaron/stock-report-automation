# ==============================================================================
# 섹션 1: 필수 라이브러리 설치
# ==============================================================================
# Colab 환경은 매번 초기화되므로, 실행 시마다 필요한 라이브러리를 설치합니다.
!pip install yfinance google-generativeai pandas

# ==============================================================================
# 섹션 2: 라이브러리 임포트 및 기본 설정
# ==============================================================================
import yfinance as yf
import google.generativeai as genai
import pandas as pd
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime
from google.colab import userdata
import time

# ==============================================================================
# 섹션 3: 보안 정보 불러오기 (API 키 및 이메일 정보)
# ==============================================================================
# [중요] 실행 전, Colab 좌측 메뉴의 '🔑(보안 비밀)'에 아래 4개 항목을 추가해야 합니다.
# 1. GOOGLE_API_KEY: 발급받은 Gemini API 키
# 2. SENDER_EMAIL: 발신자 Gmail 주소 (예: "my_email@gmail.com")
# 3. SENDER_PASSWORD: Gmail 2단계 인증 후 생성한 앱 비밀번호 (16자리)
# 4. RECEIVER_EMAIL: 리포트를 수신할 이메일 주소

try:
    GEMINI_API_KEY = userdata.get('GOOGLE_API_KEY')
    SENDER_EMAIL = userdata.get('SENDER_EMAIL')
    SENDER_PASSWORD = userdata.get('SENDER_PASSWORD')
    RECEIVER_EMAIL = userdata.get('RECEIVER_EMAIL')
    # Gemini API 설정
    genai.configure(api_key=GEMINI_API_KEY)
except Exception as e:
    print("🚨 보안 비밀 정보를 불러오는 데 실패했습니다. Colab의 '🔑' 메뉴에서 정보를 올바르게 설정했는지 확인하세요.")
    print(f"오류: {e}")


# ==============================================================================
# 섹션 4: (사용자 설정) 분석할 주식 목록 및 프롬프트 정의
# ==============================================================================
# 1. 분석을 원하는 주식 목록 (종목명과 티커)
# yfinance에서 KRX 금 현물 직접 조회가 어려워, 한국거래소의 골드선물 ETF로 대체합니다.
STOCK_LIST = {
    "마스터카드": "MA",
    "엔비디아": "NVDA",
    "필립모리스": "PM",
    "SK하이닉스": "000660.KS",
    "AMD": "AMD",
    "팔란티어": "PLTR",
    "마이크론": "MU",
    "코카콜라": "KO",
    "브로드컴": "AVGO",
    "ESPO ETF": "ESPO",
    "알파벳A": "GOOGL",
    "네이버": "035420.KS",
    "메타": "META",
    "KRX금현물(KODEX 골드선물)": "132030.KS",
    "애플": "AAPL"
}

# 2. Gemini가 사용할 프롬프트 (사용자가 제공한 프롬프트)
# 이 프롬프트는 Gemini가 따라야 할 '보고서의 형식'을 정의합니다.
USER_PROMPT_TEMPLATE = """
【정교화된 주식 분석 프롬프트】
# 최종 목표
심층 분석을 통해, 데이터에 기반한 투자 결정의 핵심 근거를 마련하고, 매수/보유/매도 전략 수립을 위한 종합 보고서 작성

# 페르소나 (Persona)
당신은 골드만삭스 소속의 시니어 애널리스트입니다. 당신의 투자 철학은 **위험 중립적(Risk-Neutral)**으로, 잠재적 손실 가능성을 통제하며 안정적인 수익 창출을 최우선으로 합니다. 당신의 분석은 감정이나 시장의 루머가 아닌, 오직 검증 가능한 데이터와 논리적 근거에 기반하며, 냉철하고 객관적인 시각을 유지합니다.
# 보고서 구조 및 분석 프롬프트
[ 보고서 최상단: 핵심 요약 및 투자 의견 ]
투자 의견 요약 (3줄)현재 상황, 핵심 투자 포인트, 주요 리스크를 각각 한 문장으로 압축하여 제시하십시오.
목표주가 및 투자의견제시 목표주가
현재 주가 ('Today()' 기준): (괴리율: X%)
투자의견: [매수 / 중립 / 매도]
매수 추천 단가와 그 이유 (얼마에 이 주식을 구매하면 좋을지?)

목표주가 산정 근거: 어떠한 밸류에이션 방법(예: PER, PBR, DCF 등)을 사용했으며, 어떠한 핵심 가정을 기반으로 목표주가를 산출했는지 명확하고 논리적으로 설명하십시오.
I. 기업 및 재무 심층 분석 (Corporate & Financial Deep Dive)
기업 개요
기업명: [기업명] ([종목코드])참고: 토스 증권 바로가기 링크 삽입
기준일자: 'Today()'
현재 주가 및 시가총액: 주가, 시가총액, 52주 최고/최저가출처: 한국거래소(KRX) 또는 NASDAQ 등 공식 거래소 명시
주요 사업: 핵심 사업 부문과 각 부문별 매출 비중을 간결하게 설명하고, 현재 사업 구성을 보여주는 원형 그래프를 제시하십시오.
재무 건전성 분석
최근 3개년 재무 요약 (표): 손익계산서, 재무상태표, 현금흐름표의 핵심 항목(매출액, 영업이익, 순이익, 자산, 부채, 자본, 영업/투자/재무 현금흐름)을 표로 요약하십시오.
재무 비율 분석:성장성: 매출액 증가율, 영업이익 증가율
수익성: 영업이익률, 순이익률, ROE(자기자본이익률)
안정성: 부채비율, 유동비율
시각 자료:[막대그래프] 최근 3년간 매출액 및 영업이익 추이
[선그래프] 최근 3년간 부채비율 및 유동비율 추이
최신 실적 및 가치 평가
분기별 실적 동향: 최근 4개 분기 매출과 영업이익을 시장 컨센서스와 비교하여 어닝 서프라이즈/쇼크 여부를 분석하십시오. (표 형식)
핵심 투자 지표:가치 지표(Valuation): PER, PBR, PSR의 의미를 간략히 설명하고, 동종 업계 평균 및 주요 경쟁사와 비교하여 현재 주가의 고평가/저평가 여부를 분석하십시오.
주당 가치 지표: EPS(주당순이익), BPS(주당순자산)의 과거 추세를 분석하십시오.
II. 시장 환경 및 경쟁 구도 분석 (Market & Competitive Landscape)
산업 분석
산업 특성: 해당 기업이 속한 산업의 성장 단계(도입기/성장기/성숙기/쇠퇴기), 경기 민감도, 주요 특징을 설명하십시오.
산업 동향 및 메가트렌드: AI, 친환경, 고령화 등 현재 시장을 관통하는 메가트렌드와 해당 산업의 연관성을 분석하고, 이것이 기회 요인인지 위협 요인인지 평가하십시오.
경쟁 분석
시장 내 위상: 주력 제품/서비스의 국내외 시장 점유율을 제시하고, 주요 경쟁사([경쟁사 A], [경쟁사 B]) 리스트를 작성하십시오.
경쟁 우위 (Economic Moat): 경쟁사와 비교하여 이 기업만이 가진 차별화된 경쟁력(기술적 해자, 브랜드 파워, 원가 우위, 네트워크 효과 등)을 구체적인 근거를 들어 분석하십시오.
시각 자료:[선그래프] 최근 1년간 [기업명]과 [경쟁사 A]의 주가 추이 비교
III. 미래 성장성 및 리스크 심층 분석 (Future Growth & Risk Analysis)
핵심 성장 동력 (Growth Drivers)
회사가 미래 성장을 위해 가장 핵심적으로 추진하는 신사업 또는 신기술([핵심 신사업/신기술명] 등)을 명시하십시오.
해당 성장 동력의 시장 잠재력, 예상 상용화 시점, 성공 시 예상되는 재무적 기여도를 분석하십시오.
정책 및 규제 영향 분석
현재 정부(국내) 또는 주요 시장(미국, EU 등)에서 논의/시행 중인 핵심 정책 및 규제(예: 기업 밸류업 프로그램, IRA, CBAM 등)를 선정하십시오.
이러한 정책/규제가 **'1. 핵심 성장 동력'**에 미칠 영향을 기회 요인과 위협 요인으로 나누어 심층 분석하십시오.
리스크 시나리오 및 민감도 분석
내부 리스크 (Micro): R&D 실패, 핵심 인력 유출, CEO 리더십 문제, 재무 구조 악화 가능성 등을 평가하십시오.
산업 리스크 (Meso): 경쟁사의 파괴적 혁신, 원자재 가격 급등, 대체재의 등장, 전방/후방 산업의 변화 등을 분석하십시오.
거시 리스크 (Macro): 금리, 환율, 인플레이션, 경기 침체 등 거시 경제 변수 변화에 대한 기업 실적의 민감도를 분석하십시오.
[신규 핵심] 리스크 대응 전략 분석 (Risk Mitigation Strategy Analysis)
위에서 식별된 주요 리스크(내부/산업/거시 각 1개씩)에 대해, 회사가 이를 인지하고 있는지, 그리고 어떻게 대응하고 있는지 분석하십시오.
분석 소스: 기업의 공식 발표 자료(사업보고서, IR 자료, 컨퍼런스 콜 녹취록), 언론 인터뷰 등을 근거로 제시하십시오.
대응 전략 평가: 회사가 제시한 대응 전략이 현실적이고 효과적인지에 대한 당신(애널리스트)의 비판적 평가를 덧붙이십시오.
IV. 최종 투자 논거 및 결론 (Investment Thesis & Conclusion)
투자 매력도 종합 평가 (Bull vs. Bear Case)
강세론 (Bull Case): 지금 당장 이 기업에 투자해야 하는 가장 매력적인 이유 3가지를 핵심 근거와 함께 제시하십시오.
약세론 (Bear Case): 투자를 주저하게 만드는 가장 치명적인 리스크 3가지를 구체적인 데이터와 함께 제시하십시오.
위험 중립적 투자자를 위한 최종 점검
투자 적합성: 분석 결과를 종합할 때, 이 기업은 '안정적 수익 추구형' 포트폴리오와 '성장주 중심형' 포트폴리오 중 어느 쪽에 더 적합합니까? 그 핵심 근거는 무엇입니까?
모니터링 지표: 만약 투자한다면, 향후 어떤 핵심 지표(예: 신사업 수주 공시, 원자재 가격, 경쟁사 신제품 출시 등)를 지속적으로 추적 관찰해야 합니까?
투자기간 관점: 단기적(1년 이내) 관점과 장기적(3년 이상) 관점에서 기대되는 핵심 모멘텀과 감수해야 할 주요 위험은 각각 무엇입니까?
V. 출력 형식 및 준수 사항
데이터 기준: 보고서에 인용되는 모든 데이터(재무, 주가, 뉴스 등)는 기준일자로부터 최근 1년 이내 자료를 우선 사용하십시오. 부득이하게 1년 이전 자료를 사용할 경우, **<span style="color:red;">(YYYY년 자료)</span>**와 같이 명확히 표기하십시오.
출처 명시: 모든 데이터와 정보는 신뢰할 수 있는 출처(DART 공시, IR 자료, 증권사 리포트, 공신력 있는 언론 등)를 기반으로 하며, 간략히 출처를 명시하십시오.
객관성 유지: 논리적 비약이나 확인되지 않은 추측은 엄격히 금지합니다. 추론이 불가피한 경우, 각주를 통해 '추론'임을 명시하고 그 근거와 한계를 서술하십시오.
시각화: 본문의 표와 그래프를 적극적으로 사용하여 가독성을 극대화하십시오. 주가 그래프는 실제 데이터를 기반으로 정확하게 작성하십시오.
전문 용어: 전문 용어 사용 시, 독자의 이해를 돕기 위해 괄호 안에 간략한 설명을 덧붙일 수 있습니다. (예: Economic Moat(경제적 해자))
"""


# ==============================================================================
# 섹션 5: 메인 실행 로직 (데이터 수집 → 리포트 생성 → 종합)
# ==============================================================================
def generate_and_send_report():
    """주식 데이터를 수집하고, 각 주식별로 리포트를 생성하여 이메일로 발송하는 메인 함수"""
    
    # --- 1. Gemini 리포트 생성을 위한 모델 설정 ---
    generation_config = {
        "temperature": 0.2, # 낮은 온도로 설정하여 일관되고 사실적인 답변 유도
        "top_p": 1,
        "top_k": 1,
        "max_output_tokens": 8192, # 방대한 분석을 위해 최대 출력 토큰을 늘림
    }
    safety_settings = [ # 유해 콘텐츠 차단 해제 (분석에 방해될 수 있음)
        {"category": "HARM_CATEGORY_HARASSMENT", "threshold": "BLOCK_NONE"},
        {"category": "HARM_CATEGORY_HATE_SPEECH", "threshold": "BLOCK_NONE"},
        {"category": "HARM_CATEGORY_SEXUALLY_EXPLICIT", "threshold": "BLOCK_NONE"},
        {"category": "HARM_CATEGORY_DANGEROUS_CONTENT", "threshold": "BLOCK_NONE"},
    ]
    model = genai.GenerativeModel(model_name="gemini-1.0-pro",
                                  generation_config=generation_config,
                                  safety_settings=safety_settings)

    # --- 2. 각 주식별로 리포트 생성 및 종합 ---
    final_report_body = ""
    print("🚀 주식 리포트 생성을 시작합니다...")
    
    for name, ticker in STOCK_LIST.items():
        print(f"\n[  분석 시작: {name} ({ticker}) ]")
        
        try:
            # --- 2-1. 데이터 수집 ---
            stock_info = yf.Ticker(ticker)
            
            # yfinance가 제공하는 최대한의 정보를 문자열로 정리
            info = stock_info.info
            data_to_analyze = f"""
# 기업명: {name} ({ticker})

## 주요 정보
- 현재가: {info.get('currentPrice') or info.get('previousClose', 'N/A')}
- 시가총액: {info.get('marketCap', 'N/A')}
- 52주 최고/최저가: {info.get('fiftyTwoWeekHigh', 'N/A')} / {info.get('fiftyTwoWeekLow', 'N/A')}
- PER: {info.get('trailingPE', 'N/A')}
- PBR: {info.get('priceToBook', 'N/A')}
- 배당수익률: {info.get('dividendYield', 'N/A')}
- 베타: {info.get('beta', 'N/A')}
- 간단한 사업 요약: {info.get('longBusinessSummary', 'N/A')}

## 재무 데이터 (최근)
- 손익계산서:\n{stock_info.financials.to_string()}
- 재무상태표:\n{stock_info.balance_sheet.to_string()}
- 현금흐름표:\n{stock_info.cashflow.to_string()}
"""
            print(f"✅ 데이터 수집 완료")

            # --- 2-2. Gemini에 보낼 최종 프롬프트 구성 ---
            final_prompt_for_gemini = f"""
아래의 원본 데이터를 사용하여, 주어진 【정교화된 주식 분석 프롬프트】의 형식에 맞춰서 '{name}'에 대한 심층 분석 보고서를 작성해 주십시오. 모든 데이터는 제공된 원본 데이터를 기반으로 하며, 형식을 엄격하게 준수해야 합니다.

--- 원본 데이터 ---
{data_to_analyze}
---

--- 보고서 형식 (반드시 이 형식을 따라주세요) ---
{USER_PROMPT_TEMPLATE}
---
"""
            # --- 2-3. Gemini API 호출하여 리포트 생성 ---
            print(f"🧠 Gemini 분석 중...")
            response = model.generate_content(final_prompt_for_gemini)
            
            # 생성된 리포트를 최종 이메일 본문에 추가
            final_report_body += f"========================================\n"
            final_report_body += f"       종목: {name} ({ticker})\n"
            final_report_body += f"========================================\n\n"
            final_report_body += response.text
            final_report_body += "\n\n\n"
            
            print(f"✅ 리포트 생성 완료!")

            # API 과호출 방지를 위해 잠시 대기
            time.sleep(10)

        except Exception as e:
            error_message = f"❌ '{name}' 분석 중 오류 발생: {e}"
            print(error_message)
            final_report_body += error_message + "\n\n"
    
    # --- 3. 종합된 리포트를 이메일로 발송 ---
    print("\n📬 최종 리포트를 이메일로 발송합니다...")
    
    today_date = datetime.now().strftime("%Y-%m-%d")
    message = MIMEMultipart("alternative")
    message["Subject"] = f"[{today_date}] 골드만삭스 스타일 주식 심층분석 리포트"
    message["From"] = SENDER_EMAIL
    message["To"] = RECEIVER_EMAIL
    
    part = MIMEText(final_report_body, "plain", "utf-8")
    message.attach(part)
    
    try:
        server = smtplib.SMTP_SSL('smtp.gmail.com', 465)
        server.login(SENDER_EMAIL, SENDER_PASSWORD)
        server.sendmail(SENDER_EMAIL, RECEIVER_EMAIL, message.as_string())
        print("✅ 이메일이 성공적으로 발송되었습니다.")
    except Exception as e:
        print(f"🚨 이메일 발송에 실패했습니다: {e}")
    finally:
        if 'server' in locals():
            server.quit()

# ==============================================================================
# 섹션 6: 스크립트 실행
# ==============================================================================
# 위에서 정의한 메인 함수를 실행합니다.
generate_and_send_report()
