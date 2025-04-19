# 1. 필요한 라이브러리 설치
!pip install transformers langchain gradio faiss-cpu sentence-transformers

# 2. 모델 및 임베딩 준비
from transformers import AutoTokenizer, AutoModelForCausalLM, pipeline
from langchain_community.llms import HuggingFacePipeline
from langchain.chains import ConversationChain
from langchain.memory import ConversationBufferMemory
from langchain.embeddings import HuggingFaceEmbeddings
from langchain_community.vectorstores import FAISS
import gradio as gr

# 3. LLM 래핑 (EXAONE-3.5-2.4B-Instruct)
model_name = "LGAI-EXAONE/EXAONE-3.5-2.4B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)
pipe = pipeline("text-generation", model=model, tokenizer=tokenizer, max_new_tokens=128)
llm = HuggingFacePipeline(pipeline=pipe)

# 4. 임베딩 및 FAISS 벡터스토어 준비
embedding_model = "sentence-transformers/all-MiniLM-L6-v2"
embeddings = HuggingFaceEmbeddings(model_name=embedding_model)

# 예시 문제 데이터 (상/중/하)
problem_data = [
    {"content": "[하] 2+3은? (A) 5 (B) 6 (C) 7", "level": "하"},
    {"content": "[중] 12/3은? (A) 2 (B) 4 (C) 6", "level": "중"},
    {"content": "[상] 소수 중 7보다 작은 것은? (A) 4 (B) 5 (C) 9", "level": "상"},
    {"content": "[하] 1+1은? (A) 1 (B) 2 (C) 3", "level": "하"},
    {"content": "[중] 15-7은? (A) 8 (B) 9 (C) 7", "level": "중"},
    {"content": "[상] 13은 소수인가요? (A) 예 (B) 아니오", "level": "상"}
]
# 문제 텍스트와 메타데이터 분리
texts = [item["content"] for item in problem_data]
metadatas = [{"level": item["level"]} for item in problem_data]

# 벡터스토어 생성
vectorstore = FAISS.from_texts(texts, embeddings, metadatas=metadatas)

# 5. 진단 문제 및 평가 로직
DIAG_QUESTIONS = [
    {"text": "[하] 2+3은? (A) 5 (B) 6 (C) 7", "answer": "A"},
    {"text": "[중] 12/3은? (A) 2 (B) 4 (C) 6", "answer": "B"},
    {"text": "[상] 소수 중 7보다 작은 것은? (A) 4 (B) 5 (C) 9", "answer": "B"}
]

def evaluate_level(responses):
    correct = sum([resp for resp in responses])
    if correct == 3:
        return "상"
    elif correct == 2:
        return "중"
    else:
        return "하"

# 6. 챗봇 클래스
class StudentTutorBot:
    def __init__(self):
        self.memory = ConversationBufferMemory()
        self.chain = ConversationChain(llm=llm, memory=self.memory)
        self.level = None

    def diagnose(self, answers):
        responses = []
        for i, q in enumerate(DIAG_QUESTIONS):
            student_answer = answers[i].strip().upper()
            is_correct =
