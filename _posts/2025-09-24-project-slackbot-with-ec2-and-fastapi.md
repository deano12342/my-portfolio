---
layout: single
title: "AWS EC2와 FastAPI로 DB 자료 검색 슬랙봇 만들기 🤖"
date: 2025-09-24 15:30:00 +0900
categories: [project, aws, python]
tags: [aws, ec2, fastapi, python, openai, pinecone, slack, chatbot, portfolio]
---

## 1. 들어가며 (Introduction)

"이 내용, 예전에 보고된 적 있던가?"

팀 동료의 이 질문 하나에서 프로젝트가 시작되었습니다. 저는 PR 팀에서 근무하고 있어, 저희 팀에서는 과거에 보고되었던 자사 관련 기사들을 다시 찾아봐야 하는 일이 잦았습니다. 이 때문에 매번 담당자가 암호화된 파일들을 뒤져가며 수동으로 히스토리 파일을 검색하는 비효율이 발생했습니다. 이 반복적인 업무를 자동화하고, 축적된 데이터를 자산으로 활용하기 위해 AI 챗봇 개발에 도전했습니다.

완성된 챗봇은 슬랙에서 간단한 명령어로 특정 키워드가 포함된 과거 기사 내용을 요약, 검색해주는 역할을 수행합니다.

본 게시글에서는 '기보고 기사 데이터'를 찾는 챗봇으로 활용했으나, 업무에 맞게 다양하게 응용해 활용하시는 것을 추천합니다.
**구축 과정에서 AI의 조언을 구하는 것을 주저하지 마세요!** 모든 코드 작성에는 AI의 보조가 있었습니다.


![챗봇 가동 화면](/assets/img/chat-bot-example.gif)

<br>

### 💡 핵심 아키텍처 및 기술 스택

이번 프로젝트는 다음과 같은 아키텍처와 기술 스택으로 구현했습니다.
가능한 무료로 사용할 수 있는 프로그램으로 구성됐고, 사용하시다가 필요시 특정 프로그램(클라우드 스펙 등)을 업그레이드 하실 것을 권장합니다.

*가장 먼저 진행되어야 하는 부분: 바탕화면에 프로젝트 폴더를 만들어주세요.(제목 e.g: chat-bot)*

- **Architecture(작동 구조)**: Slack → AWS EC2 (FastAPI) → Pinecone ↔ OpenAI → Slack
- **클라우드 인프라(AWS)**: **AWS EC2 (Free Tier)**
- **Backend Framework**: **FastAPI**
- **AI/ML**: **OpenAI API** (Embedding & Completion), **Pinecone API** (Vector DB)
- **Language & Tools**: **Python**, Anaconda, PuTTY, FileZilla

위 프로그램, 시스템 들은 모두 간단한 검색을 통해 다운로드 및 설치가 가능합니다.
구축 과정 중에 하나 하나 설치하셔도 무방하지만, 가능한 API 코드 등은 미리 확보 후 진행하시는 것이 수월할 것으로 생각됩니다.

<br>
---

## 2. STEP 1: 흩어진 데이터 길들이기 (데이터 전처리)

가장 먼저 해결해야 할 과제는 AI가 학습할 데이터를 준비하는 것이었습니다. 기존 기사 DB는 여러 `시트(Sheet)`로 월별, 분기별 내용이 흩어져 있는 `.xlsm` 엑셀 파일이었습니다. 이대로는 AI가 데이터를 읽고 학습하기 어려웠죠. AI는 `.csv` 파일 등 특정 텍스트 형식의 데이터만 제대로 파악할 수 있기 때문이었습니다.

이 문제를 해결하기 위해 두가지 과정을 거쳤습니다. 첫번째는 엑셀 매크로(VBA)를 활용하는 방법입니다. 엑셀 파일의 모든 시트를 한 번에 각각의 CSV 파일로 추출하는 방법이 있습니다. 엑셀 자체 기능인 매크로(VBA)를 이용하는 것이 가장 확실하고 간단합니다. 파이썬으로도 가능하지만, 이미 매크로가 포함된 파일이니만큼 엑셀의 기능을 활용하는 것이 더 직관적일 수 있습니다.

**제가 변환한 파일의 경우 날짜별 데이터가 시트별 구분되어 있었고, '시트명'에 '날짜'가 들어가 있었습니다.**
즉, 저는 날짜를 제목으로 가지는 CSV 파일을 확보하고자 본 매크로를 사용한 것입니다.
각자의 목표에 맞게 AI에게 의뢰해 매크로를 수정하시길 권장드립니다.


*사용 프롬프트: "여러개의 시트(날짜별로)에 로우데이터들이 들어가 있는데, 이걸 양식에 맞춰 CSV 파일로 바꾸고 싶어. VBA 매크로를 만들어줘."*

1. 엑셀 파일을 열어주세요.
2. VBA 편집기 열기: 키보드에서 Alt + F11 키를 동시에 눌러 VBA 편집기 창을 엽니다.
3. 모듈 삽입: VBA 편집기 왼쪽 상단의 '프로젝트 탐색기'에서 현재 엑셀 파일 이름을 찾습니다. (예: VBAProject (25년_기사 DB...))
4. 해당 파일 이름 위에서 마우스 오른쪽 버튼을 클릭하고 **삽입(Insert) > 모듈(Module)**을 선택합니다. 그럼 오른쪽에 하얀색 빈 코드 창이 나타납니다.
5. 코드를 통째로 입력해주세요. (아래로 예시)


```VB.Net
Sub SaveAllSheetsAsCSV()
    ' 에러가 발생해도 계속 진행하도록 설정
    On Error Resume Next

    Dim ws As Worksheet
    Dim wbPath As String
    Dim safeName As String

    ' CSV 파일이 저장될 경로를 현재 엑셀 파일이 있는 폴더로 지정
    wbPath = ThisWorkbook.Path & "\"

    ' 사용자에게 마지막으로 확인 메시지를 보여줌
    If MsgBox("총 " & ThisWorkbook.Worksheets.Count & "개의 시트를 CSV로 저장하시겠습니까?", vbYesNo) = vbNo Then Exit Sub

    ' 엑셀의 모든 시트를 하나씩 순회
    For Each ws In ThisWorkbook.Worksheets
        ' 파일 이름에 사용할 수 없는 특수문자 제거
        safeName = ws.Name
        safeName = Replace(safeName, "/", "-")
        safeName = Replace(safeName, "\", "-")
        safeName = Replace(safeName, ":", "-")
        safeName = Replace(safeName, "*", "-")
        safeName = Replace(safeName, "?", "-")
        safeName = Replace(safeName, """", "'")
        safeName = Replace(safeName, "<", "-")
        safeName = Replace(safeName, ">", "-")
        safeName = Replace(safeName, "|", "-")

        ' 현재 시트를 CSV 형식으로 저장 (파일 이름은 시트 이름과 동일)
        ws.SaveAs Filename:=wbPath & safeName & ".csv", FileFormat:=xlCSVUTF8, CreateBackup:=False
    Next ws

    On Error GoTo 0
    MsgBox "작업 완료! 모든 시트가 엑셀 파일과 같은 폴더에 CSV 파일로 저장되었습니다.", vbInformation
End Sub
```

두 번째로는 **낱장으로 변환된 CSV 파일들을 통일된 양식을 가진 하나의 CSV 파일로 합치는 것** 입니다.
아래는 제가 실제로 작성한 자동 데이터 변환 파이썬 코드입니다. 이를 얻기 위해 AI(제미나이)에게 전달한 프롬프트는 아래와 같습니다.

사용 프롬프트: *이 csv 파일들을 내가 원하는 형식에 맞춰 변환하고 취합하는 py 코드를 작성해줘.* (아래로 코드 예시)


```python
# 데이터 취합기.py
import pandas as pd
import os
import glob
import re

# --- 사용자 설정 ---
output_filename = '최종_결과물.csv'
# --- 설정 끝 ---

def process_csv_file(df, date_from_filename):
    """CSV 파일과 날짜를 받아, 이미지 분석 기반 최종 로직으로 데이터를 파싱하는 함수"""
    articles_list = []
    
    # 데이터프레임의 모든 행을 인덱스를 사용해 순회
    for i in range(len(df)):
        # 첫 번째 열의 값이 비어있으면 건너뛰기
        if pd.isna(df.iloc[i, 0]):
            continue
            
        label = str(df.iloc[i, 0]).strip()

        # '○' 기호를 포함하면 새로운 기사의 시작으로 판단
        if '○' in label:
            try:
                # 1. 제목 추출
                title = re.sub(r'^[└○\s]+', '', label).strip()
                
                # 기본값 설정
                link = ''
                content = ''

                # 2. 링크 추출: 제목 바로 다음 행(i+1)의 첫 번째 칸(A열)을 확인
                if i + 1 < len(df) and pd.notna(df.iloc[i + 1, 0]):
                    potential_link = str(df.iloc[i + 1, 0]).strip()
                    if potential_link.startswith('http'):
                        link = potential_link

                # 3. 내용 추출: 링크 다음 행(i+2)의 두 번째 칸(B열)을 확인
                if i + 2 < len(df) and pd.isna(df.iloc[i + 2, 0]) and len(df.columns) > 1 and pd.notna(df.iloc[i + 2, 1]):
                    content = str(df.iloc[i + 2, 1]).strip()
                
                # 수집한 정보로 딕셔너리 생성
                current_article = {
                    'title': title,
                    'link': link,
                    'date': date_from_filename,
                    'content': content
                }
                articles_list.append(current_article)

            except Exception:
                # 예상치 못한 오류가 발생해도 건너뛰도록 처리
                continue
                
    return pd.DataFrame(articles_list)

# --- 메인 코드 실행 ---
all_csv_files = glob.glob('*.csv')
all_dataframes = []

print(f"총 {len(all_csv_files)}개의 CSV 파일 변환을 시작합니다...")

date_pattern = re.compile(r'(\d{4}-\d{2}-\d{2})')

for file_path in all_csv_files:
    if file_path == output_filename:
        continue
    
    try:
        match = date_pattern.search(file_path)
        if not match:
            print(f"  -> 경고: '{file_path}' 파일 이름에서 날짜를 찾을 수 없어 건너뜁니다.")
            continue
        
        date_str = match.group(1).replace('-', '/')
        
        print(f"- '{file_path}' 파일 처리 중 (날짜: {date_str})...")
        try:
            df = pd.read_csv(file_path, header=None, encoding='utf-8')
        except UnicodeDecodeError:
            df = pd.read_csv(file_path, header=None, encoding='cp949')
            
        processed_df = process_csv_file(df, date_str)
        if not processed_df.empty:
            all_dataframes.append(processed_df)
            
    except Exception as e:
        print(f"  -> 오류: '{file_path}' 파일을 처리하는 중 문제가 발생했습니다: {e}")

if all_dataframes:
    final_df = pd.concat(all_dataframes, ignore_index=True)
    final_df = final_df.reindex(columns=['title', 'link', 'date', 'content'])
    final_df.to_csv(output_filename, index=False, encoding='utf-8-sig')
    
    print(f"\n🎉 작업 완료! 총 {len(final_df)}개의 기사 데이터를 성공적으로 변환하여")
    print(f"'{output_filename}' 파일로 저장했습니다.")
else:
    print("\n[최종 확인 필요] 처리할 데이터를 찾지 못했습니다.")
```

이 스크립트 덕분에 복잡했던 데이터 정제 작업을 자동화하고, 일관된 형식의 학습 데이터를 확보할 수 있었습니다.

<br>
___

## 3. STEP 2: 챗봇의 뇌와 기억력 만들기, CSV 데이터 학습시키기 (AI 모델 & 벡터 DB)

단순히 OpenAI의 ChatGPT API만 사용하면 저희가 가진 내부 데이터에 대해 답변할 수 없습니다. 이 문제를 해결하기 위해 **RAG (Retrieval-Augmented Generation)** 아키텍처를 도입하고, `Pinecone`을 챗봇의 '기억 장치'로 활용했습니다.

1.  **데이터 임베딩 (Embedding)**: 전처리한 `.csv` 파일의 텍스트(기사 내용)를 **OpenAI의 Embedding API**를 통해 벡터(Vector), 즉 AI가 이해할 수 있는 숫자 배열로 변환합니다.
2.  **벡터 DB 구축 (Indexing)**: 변환된 벡터 데이터들을 고유 ID와 함께 **Pinecone**에 저장합니다. 이제 챗봇은 수많은 기사 내용을 빠르게 검색할 수 있는 '기억력'을 갖게 되었습니다.

### ⚙️ CSV 데이터를 Pinecone에 학습(인덱싱)시키기

이제 실제로 앞서 생성한 `.csv` 파일의 데이터를 Pinecone에 저장하는 스크립트를 작성할 차례입니다. 이 작업은 챗봇 서버와는 별개로, 데이터를 DB에 미리 넣어두기 위해 **한 번만 실행**하는 스크립트입니다.

프로젝트 폴더에 `index_data.py`와 같이 새로운 파이썬 파일을 만들고 아래 코드를 작성해주세요.

```python
# index_data.py
import os
import pandas as pd
import openai
import pinecone
from dotenv import load_dotenv
from tqdm import tqdm # 진행 상황을 시각적으로 보여주는 라이브러리

# .env 파일에서 API 키 로드
load_dotenv()
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
PINECONE_API_KEY = os.getenv("PINECONE_API_KEY")

# --- API 클라이언트 초기화 ---
openai.api_key = OPENAI_API_KEY
pinecone.init(api_key=PINECONE_API_KEY, environment="gcp-starter") # Pinecone 환경명은 본인에 맞게 수정
pinecone_index = pinecone.Index("articles") # Pinecone 인덱스 이름

# --- 데이터 로드 ---
DATA_PATH = "path/to/your/processed_articles.csv" # STEP 1에서 생성한 CSV 파일 경로
df = pd.read_csv(DATA_PATH)

print("데이터 인덱싱을 시작합니다...")

# --- Pinecone에 데이터 저장(Upsert) ---
# 데이터를 100개씩 묶어서 처리 (API 효율성 증대)
batch_size = 100
for i in tqdm(range(0, len(df), batch_size)):
    i_end = min(i + batch_size, len(df))
    batch = df.iloc[i:i_end]
    
    # 임베딩할 텍스트 추출 (예: '본문' 컬럼)
    texts_to_embed = batch['본문'].tolist()
    
    # OpenAI Embedding API 호출
    res = openai.Embedding.create(input=texts_to_embed, model="text-embedding-ada-002")
    embeds = [record['embedding'] for record in res['data']]
    
    # Pinecone에 저장할 데이터 형식으로 가공
    to_upsert = []
    for idx, row in batch.iterrows():
        vector_id = f"article_{idx}" # 각 데이터의 고유 ID 생성
        metadata = {"text": row['본문'], "title": row['기사제목']} # 검색 시 함께 반환될 정보
        to_upsert.append((vector_id, embeds[idx - i], metadata))
    
    # Pinecone에 최종 저장
    pinecone_index.upsert(vectors=to_upsert)

print("데이터 인덱싱이 완료되었습니다.")
```

이 스크립트를 터미널에서 python index_data.py 명령어로 실행하면, CSV 파일의 모든 기사 내용이 벡터로 변환되어 Pinecone DB에 차곡차곡 저장됩니다. 이제 우리의 챗봇은 이 '기억 장치'를 자유롭게 검색하여 질문에 대한 근거를 찾을 수 있게 되었습니다.

<br>
___

## 4. STEP 3: 챗봇의 몸통 만들기 (FastAPI 서버 구축)

이제 Slack과 AI 모델을 연결해 줄 API 서버를 만들 차례입니다. 가볍고 빠른 **FastAPI**를 사용해 Slack의 요청을 받아 처리하고 응답을 보내주는 '몸통'을 구축했습니다.

프로젝트 폴더(chat-bot 폴더)에 `main.py`라는 이름으로 파이썬 파일을 생성하고, 아래의 전체 코드를 붙여넣어 주세요. 이 파일 하나가 API 서버의 모든 로직을 담당하게 됩니다.

아래 코드의 주요 역할은 다음과 같습니다.
- Slack API 연동 시 처음 한 번 필요한 **URL 검증(`challenge`)** 요청에 응답합니다.
- 사용자가 봇을 멘션하며 입력한 **`/기사체크` 명령어**를 감지하고 검색 키워드를 추출합니다.
- 추출된 키워드를 바탕으로 **OpenAI와 Pinecone API를 호출**하여 최종 답변을 생성합니다.
- 생성된 답변을 다시 **Slack 채널로 전송**합니다.

```python
# main.py
import os
import openai
import pinecone
from fastapi import FastAPI, Request, Response
from slack_sdk import WebClient

# --- 초기 설정: API 키는 보안을 위해 환경 변수에서 불러옵니다.
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
PINECONE_API_KEY = os.getenv("PINECONE_API_KEY")
SLACK_BOT_TOKEN = os.getenv("SLACK_BOT_TOKEN")

# --- API 클라이언트 초기화
openai.api_key = OPENAI_API_KEY
pinecone.init(api_key=PINECONE_API_KEY, environment="gcp-starter")
slack_client = WebClient(token=SLACK_BOT_TOKEN)
pinecone_index = pinecone.Index("articles")

app = FastAPI()

# --- FastAPI 엔드포인트 ---
@app.post("/slack/events")
async def slack_events(request: Request):
    body = await request.json()
    
    # Slack의 URL 검증(challenge) 요청에 대한 응답 처리
    if "challenge" in body:
        return {"challenge": body["challenge"]}
        
    # 실제 슬랙 이벤트 처리
    event = body.get("event", {})
    channel_id = event.get("channel")
    user_text = event.get("text", "")

    # 봇을 멘션하고 '/기사체크' 명령어가 포함된 경우에만 동작
    if event.get("type") == "app_mention" and "/기사체크" in user_text:
        keyword = user_text.split("/기사체크")[-1].strip()

        if not keyword:
            slack_client.chat_postMessage(channel=channel_id, text="검색할 키워드를 입력해주세요. (예: /기사체크 AI)")
            return Response(status_code=200)

        # 1. 사용자 키워드를 벡터로 변환 (by OpenAI)
        query_vector = openai.Embedding.create(input=[keyword], model="text-embedding-ada-002")['data'][0]['embedding']

        # 2. Pinecone에서 유사한 기사 내용 검색
        search_results = pinecone_index.query(vector=query_vector, top_k=3, include_metadata=True)
        
        # 3. 검색 결과를 바탕으로 OpenAI에 보낼 프롬프트 구성
        context = ""
        for match in search_results['matches']:
            context += match['metadata']['text'] + "\n\n"

        prompt = f"""
        You are a helpful assistant that summarizes past news articles.
        Based on the context below, please answer the user's question.
        ---
        Context: {context}
        ---
        Question: {keyword}
        """

        # 4. 구성된 프롬프트로 OpenAI에 답변 생성 요청
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": prompt}]
        )
        answer = response.choices[0].message.content

        # 5. 생성된 답변을 Slack으로 전송
        slack_client.chat_postMessage(channel=channel_id, text=answer)

    return Response(status_code=200)
```

**⚙️ 로컬 환경에서 챗봇 구동 테스트하기**
AWS에 배포하기 전, 코드가 내 컴퓨터(로컬)에서 잘 작동하는지 확인하는 과정입니다.

**1. API 키 설정 (필수)**
main.py는 환경 변수에서 API 키를 읽어옵니다. 로컬 테스트를 위해, main.py와 같은 폴더에 .env 파일을 만들고 아래와 같이 키를 입력해주세요. (python-dotenv 라이브러리 설치 필요: pip install python-dotenv)

```
# .env 파일 내용
OPENAI_API_KEY="sk-..."
PINECONE_API_KEY="..."
SLACK_BOT_TOKEN="xoxb-..."
```

그리고 main.py 상단에 from dotenv import load_dotenv 와 load_dotenv() 코드를 추가해주면 .env 파일에서 키를 읽어올 수 있습니다.

**2. FastAPI 서버 실행**
터미널(아나콘다 프롬프트 등)을 열고 main.py 파일이 있는 위치로 이동한 뒤, 아래 명령어를 입력하여 서버를 실행합니다.

```
uvicorn main:app --reload
```

**3. curl로 가짜 슬랙 요청 보내기**
이제 다른 터미널 창을 열고, curl 명령어를 사용해 실제 슬랙에서 오는 것과 유사한 가짜 요청을 우리 로컬 서버(http://127.0.0.1:8000)로 보내봅니다.

```
curl -X POST -H "Content-Type: application/json" \
-d '{"event": {"type": "app_mention", "text": "<@BOT_ID> /명령어 AI"}}' \
[http://127.0.0.1:8000/slack/events](http://127.0.0.1:8000/slack/events)
```

**4. 결과 확인**
curl 명령어를 실행했을 때, FastAPI 서버를 실행시킨 터미널에 "명령어 감지! 처리 시작..." 이나 "생성된 답변: ..." 같은 로그가 나타나면 성공입니다. 이는 우리 서버가 가짜 요청을 잘 받아서 OpenAI와 Pinecone API까지 정상적으로 호출했다는 의미입니다.

이제 코드가 잘 작동하는 것을 확인했으니, 다음 단계인 AWS 배포로 넘어갈 준비가 되었습니다.

<br>

## 5. STEP 4: 외부 서버와 연결하기 (AWS EC2 배포)

로컬에서 완성된 FastAPI 서버를 이제 24시간 동작하는 **AWS EC2** 서버에 배포할 차례입니다.

1.  **EC2 인스턴스 생성**: AWS 콘솔에서 **프리티어(t2.micro)** 사양의 Ubuntu 인스턴스를 생성합니다.
2.  **서버 접속 (PuTTY)**: AWS에서 발급받은 `.pem` 키를 **PuTTYgen**으로 `.ppk` 파일로 변환한 뒤, **PuTTY**를 사용해 EC2 서버에 SSH로 접속합니다.
3.  **파일 업로드 (FileZilla 프로그램 사용)**: 로컬 PC의 FastAPI 프로젝트 파일들을 **FileZilla**를 이용해 EC2 서버의 홈 디렉토리로 업로드합니다.
4.  **서버 실행**: EC2 서버 터미널에서 `uvicorn`을 사용해 FastAPI 앱을 실행했습니다. 터미널을 종료해도 서버가 꺼지지 않도록 `nohup` 명령어를 함께 사용했습니다.
    ```bash
    nohup uvicorn main:app --host 0.0.0.0 --port 8000 &
    ```
5.  **방화벽 설정 (보안 그룹)**: **가장 중요한 단계**입니다. 외부(Slack)에서 EC2 서버의 8000번 포트로 접근할 수 있도록 **AWS EC2 보안 그룹**의 **인바운드 규칙**에 `사용자 지정 TCP`, `포트: 8000`, `소스: 0.0.0.0/0` 규칙을 추가해주었습니다.

___
<br>

## 6. 🔥 시행착오와 해결 과정 (Troubleshooting)

프로젝트는 이런 저런 오류로 인해 어려움을 겪었는데요. 혹시 같은 경험을 하실 분들을 위해 제가 겪었던 주요 문제와 해결 과정을 공유합니다.

### API 키 지옥 탈출기
처음에는 코드에 OpenAI, Pinecone, Slack API 키를 그대로 하드코딩했습니다. 하지만 이는 보안상 매우 취약한 방법입니다. GitHub에 코드를 올리기라도 하면 키가 그대로 노출되는 상황이 발생할 수 있으므로 주의가 필요합니다.


**해결책**: `os.getenv()` 함수를 사용해 시스템 **환경 변수**에서 API 키를 읽어오도록 코드를 수정했습니다. EC2 서버에서는 `.env` 파일을 만들어 환경 변수를 관리했고, 이 파일은 `.gitignore`에 추가하여 Git에 올라가지 않도록 처리했습니다.

<br>

### Slack 권한과의 씨름: 'missing_scope' 오류 해결하기
분명 코드는 완벽한데 챗봇이 아무런 반응이 없거나, 로그에 `missing_scope` 라는 오류가 찍히는 경우가 있었습니다. 이는 우리 챗봇 앱이 Slack 워크스페이스 내에서 특정 행동을 할 수 있는 권한을 부여받지 못했기 때문입니다. Slack은 보안을 위해 각 앱에 필요한 최소한의 권한만 부여하도록 설계되어 있습니다.

우리 챗봇이 정상적으로 작동하기 위해서는 최소한 두 가지 권한이 필요합니다.

- **`app_mentions:read`**: 채널에서 누군가 봇을 `@멘션`했을 때, 그 메시지를 읽을 수 있는 권한입니다. 이 권한이 없으면 봇은 자신이 호출되었다는 사실조차 알 수 없습니다.
- **`chat:write`**: 처리된 결과를 다시 채널에 메시지로 포스팅할 수 있는 권한입니다. 이 권한이 없으면 봇은 답변을 생성하고도 답을 할 수 없습니다.

**해결 방법**:
1.  Slack API 페이지에서 만든 앱 설정으로 이동합니다.
2.  왼쪽 메뉴에서 **OAuth & Permissions**를 클릭합니다.
3.  페이지를 아래로 스크롤하여 **Scopes** 섹션으로 이동합니다.
4.  **Bot Token Scopes** 아래의 `Add an OAuth Scope` 버튼을 클릭하여 `app_mentions:read` 와 `chat:write` 를 각각 추가해줍니다.
5.  **가장 중요한 단계**: 스코프를 변경한 후에는 페이지 상단에 나타나는 노란색 안내창에서 **`reinstall your app` 링크를 클릭하여 워크스페이스에 앱을 다시 설치**해주어야 변경된 권한이 적용됩니다.

<br>

### 살아있는 DB, 지속적인 데이터 업데이트
챗봇을 완성하고 나니 새로운 질문이 생겼습니다. "앞으로 추가되는 기사는 어떻게 학습시켜야 하지?"

**해결책**:

-   **단기적 해결**: 새로운 데이터가 포함된 `.csv` 파일을 EC2에 다시 업로드하고, 데이터 임베딩 및 Pinecone 업로드 스크립트를 수동으로 재실행하는 방법을 택했습니다.

-   **장기적 개선 방향**: 이 과정을 자동화하기 위해, 매일 특정 시간에 스크립트를 실행하는 **`cron` 스케줄러**를 EC2에 설정하거나, 파일 업로드용 API 엔드포인트를 FastAPI에 추가하는 방안을 다음 개선 과제로 남겨두었습니다.

<br>
___

## 7. 마치며 (Conclusion)

이번 프로젝트를 통해 단순히 라이브러리를 사용하는 것을 넘어, 데이터 전처리부터 클라우드 배포, 그리고 운영까지 이어지는 전체 파이프라인을 직접 경험해볼 수 있었습니다. 
반복적인 업무를 자동화하는 작은 아이디어에서 시작했지만, 그 과정에서 얻은 효율 및 역량 성장은 기대 이상이었습니다.

긴 글 읽어주셔서 감사합니다.