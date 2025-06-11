# granite-7b-lab 모델을 사용하여 sys_symm page에 chatbot 삽입

## 목차

* [1] pip package 설치
* [2] `llm/llm_func.py` 추가
* [3] `routes/chat.py` 추가
* [4] `app/__init__.py` 수정
* [5] `app/templates/contents/sys_summ.html` 수정
* [6] `nmon-wrangler.py` 수정

```bash
/nmonwrangler/
├── app/
│   ├── routes/
│   │   └── chat.py
│   ├── templates/
│   │   └── contents/
│   │       └── sys_summ.html
│   ├── nmon_wrangler.py
│   └── __init__.py
├── llm/
    └── llm_func.py
```

## 과정
* **[1] pip package 설치**


```bash
(venv) MacBook-Air:NmonWrangler junghyuncha$ pip list
Package                   Version
------------------------- ------------
altgraph                  0.17.4
blinker                   1.7.0
click                     8.1.7
Flask                     3.0.0
importlib-metadata        7.0.1
itsdangerous              2.1.2
Jinja2                    3.1.2
macholib                  1.16.3
MarkupSafe                2.1.3
numpy                     1.26.2
packaging                 23.2
pandas                    2.1.4
pip                       24.0
pyinstaller               6.3.0
pyinstaller-hooks-contrib 2024.0
python-dateutil           2.8.2
pytz                      2023.3.post1
setuptools                69.2.0
six                       1.16.0
tzdata                    2023.4
tzlocal                   5.2
Werkzeug                  3.0.1
zipp                      3.17.0
```

```bash
(venv) MacBook-Air:NmonWrangler junghyuncha$ pip install llama-cpp-python
```

* **[2] llm/llm_func.py 추가**
- model download(hugging face에서 download 하였습니다. .gguf 당 4GB)
- model 을 flask에서 사용 할 수 있도록 driver download(llama-cpp-python 라이브러리를 사용하여 Flask 애플리케이션에서 모델을 호출하는 API를 만들기)
- 목적 : LLM 관련 로직 분리 (모델 로딩 및 ask_llm() 함수 등)
- `llm_func.py` 파일에 아래 코드 추가

```python
from llama_cpp import Llama

# 최초 1회 로딩만 되도록 전역에서 호출
class llm():
    def __init__(self, model_path):
        self.llm = Llama(model_path=model_path)

    def ask_llm(self, prompt):

        tuned_prompt = f"""<s>
        <|system|>
        You are a helpful assistant.

        <|user|>
        {prompt}

        <|assistant|>
        """

        response = self.llm(prompt=tuned_prompt, max_tokens=256, stop=["<|endoftext|>"])
        return response["choices"][0]["text"].strip()
```

* **[3] routes/chat.py 추가**
- chat.py: /chat 엔드포인트를 정의하여 사용자의 질문을 받아 모델의 응답을 생성
- 모델의 설치 위치를 명시해야함.(local에 model이 설치 되어 있어야함.)
- `chat.py` 파일에 아래 코드 추가

```python
from flask import Blueprint, request, jsonify
from llm import llm_func

chat_bp = Blueprint('chat', __name__)
llm = llm_func.llm("/Users/junghyuncha/granite-7b-lab-GGUF/granite-7b-lab-Q4_K_M.gguf")

@chat_bp.route("/chat", methods=["POST"])
def chat():
    data = request.get_json()
    message = data.get("message", "")
    history = data.get("history", []) 

    if not message:
        return jsonify({"error": "No message provided"}), 400
    try:
        formatted = "<s>\n<|system|>\nYou are a helpful assistant.\n"
        for item in history:
            if item["role"] == "user":
                formatted += f"<|user|>\n{item['message']}\n"
            elif item["role"] == "bot":
                formatted += f"<|assistant|>\n{item['message']}<|endoftext|>\n"
        formatted += f"<|user|>\n{message}\n<|assistant|>\n"

        answer = llm.ask_llm(prompt=formatted)
        return jsonify({"response": answer})
    except Exception as e:
        return jsonify({"error": str(e)}), 500
```

* **[4] app/__init__.py 수정**
- `__init__.py` 파일에 아래 코드 추가

```python
...
from app.routes.chat import chat_bp
...
def create_app():
    ...
    # llm 사용을 위해 추가
    app.register_blueprint(chat_bp)
    ...
```

* **[5] app/templates/contents/sys_summ.html 수정**
- chatbot의 말풍선 색, 위치 등 설정
- `sys_summ.html` 파일에 아래 코드 추가

```html
...
<div class="container">
    <style>
        #chatbox {
            height: 200px;
            overflow-y: auto;
            padding: 10px;
            border: none;
            background-color: transparent; 
        }
            
        .chat-line {
            display: flex;
            margin: 8px 0;
        }
            
        .chat-line.user {
            justify-content: flex-end;
        }
            
        .chat-line.bot {
            justify-content: flex-start;
        }
            
        .chat-bubble {
            max-width: 75%;
            padding: 10px 14px;
            border-radius: 14px;
            font-size: 0.95rem;
            line-height: 1.4;
            white-space: pre-wrap;
            word-wrap: break-word;
            border: none;
            box-shadow: none;
        }
            
        .chat-bubble.user {
            background-color: #d0ebff;
            border-bottom-right-radius: 0;
            text-align: right;
        }
            
        .chat-bubble.bot {
            background-color: #e9ecef;
            border-bottom-left-radius: 0;
            text-align: left;
        }
            
        #userInput {
            flex: 1;
        }
    </style>
    <div class="row mx-5">
        <hr>
        <p>Note the the avg/max values for User%, Sys%, Wait% and Idle% are independent and will not add up to 100%. The CPU% column shows avg/max values for the sum of usr%+sys% during each interval. For micro-partitions, the values are shown as a percentage of the Virtual Processors – they do not relate to the CPU_ALL values.</p>
        <p>For non-partitioned or dedicated CPU partitions the graph shows the total CPU Utilization (%usr + %sys) together with the Disk I/O rate (taken from the DISKXFER sheet) by time of day. For micro-partitions the graph shows the number of physical CPUs being used instead of CPU%.</p>
        <p>The value “Max:Avg” is simply the maximum value divided by the average. If monitored over a long period of time the value for CPU% can be useful in spotting a system reaching saturation level (the ratio will steadily decrease).</p>
        <hr>
        <div id="chatbot" class="p-3">
            <div class="text-center w-100">
                <h5><b>AI Chat Assistant</b></h5>
            </div>
            <div id="chatbox"></div>
            <div class="d-flex mt-2">
                <input type="text" id="userInput" class="form-control me-2" placeholder="Type your question...">
                <button onclick="sendMessage()" class="btn btn-primary">Send</button>
            </div>
        </div>     
...
{% block scripts %}
<!--Load the AJAX API-->
<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
<script type="text/javascript">
    
    // Load the Visualization API and the corechart package.
    google.charts.load('current', {'packages':['corechart', 'controls']});

    // Set a callback to run when the Google Visualization API is loaded.
    google.charts.setOnLoadCallback(drawChart);

    // Callback that creates and populates a data table,
    // instantiates the pie chart, passes in the data and
    // draws it.

    // 아래부터 추가
    let chatHistory = [];

    function renderChat() {
        const chatbox = document.getElementById('chatbox');
        chatbox.innerHTML = ''; // Clear existing messages

        for (const entry of chatHistory) {
            const line = document.createElement('div');
            line.className = entry.role === 'user' ? 'chat-line user' : 'chat-line bot';

            const bubble = document.createElement('div');
            bubble.className = entry.role === 'user' ? 'chat-bubble user' : 'chat-bubble bot';
            bubble.innerHTML = `<b>${entry.role === 'user' ? 'You' : 'Bot'}:</b> ${entry.message}`;

            line.appendChild(bubble);
            chatbox.appendChild(line);
        }

        chatbox.scrollTop = chatbox.scrollHeight;
    }

    function sendMessage() {
        const input = document.getElementById('userInput');
        const message = input.value.trim();
        if (!message) return;

        chatHistory.push({ role: 'user', message });
        renderChat();
        input.value = '';

        fetch('/chat', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ message, history: chatHistory })  // 함께 전송
        })
        .then(res => res.json())
        .then(data => {
            chatHistory.push({ role: 'bot', message: data.response });
            renderChat();
        })
        .catch(err => {
            chatHistory.push({ role: 'bot', message: `Error: ${err}` });
            renderChat();
        });
    }
...
```

* **[6] nmon-wrangler.py 수정**
- `nmon_wrangler.py` 파일에 아래 코드 추가
- 모두 추가 후 `nmon_wrangler.py`에서 run 하여 chatbot 테스트 진행

```python
...
from llm import llm_func
...
```