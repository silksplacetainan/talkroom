<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <!-- 防縮放設定，確保手機按下按鈕時不會放大畫面 -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>網頁對講機</title>
    <!-- 引入 Firebase SDK -->
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>
    
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #1a1a2e;
            color: white;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            overflow: hidden;
        }
        .container {
            text-align: center;
            width: 100%;
            max-width: 400px;
            padding: 20px;
        }
        #login-screen, #ptt-screen {
            display: flex;
            flex-direction: column;
            align-items: center;
            width: 100%;
        }
        #ptt-screen { display: none; }
        
        input {
            width: 80%;
            padding: 15px;
            margin: 10px 0;
            border-radius: 8px;
            border: none;
            font-size: 16px;
            text-align: center;
        }
        .btn-join {
            background-color: #0f3460;
            color: white;
            border: none;
            padding: 15px 30px;
            border-radius: 8px;
            font-size: 18px;
            cursor: pointer;
            margin-top: 10px;
            width: 80%;
        }
        /* 大圓形 PTT 按鈕 */
        #ptt-btn {
            width: 180px;
            height: 180px;
            border-radius: 50%;
            background-color: #e94560;
            border: 8px solid #333;
            color: white;
            font-size: 24px;
            font-weight: bold;
            cursor: pointer;
            box-shadow: 0 10px 20px rgba(0,0,0,0.5);
            transition: all 0.1s;
            display: flex;
            align-items: center;
            justify-content: center;
            user-select: none;
            -webkit-user-select: none;
        }
        #ptt-btn:active {
            background-color: #d1304c;
            transform: scale(0.95);
            box-shadow: 0 5px 10px rgba(0,0,0,0.5);
        }
        /* 說話時的按鈕狀態 */
        #ptt-btn.speaking {
            background-color: #4caf50;
            border-color: #2e7d32;
            box-shadow: 0 0 30px #4caf50;
        }
        #status-text { margin-top: 20px; font-size: 18px; color: #ccc; }
        .room-title { font-size: 24px; margin-bottom: 30px; color: #e94560;}
    </style>
</head>
<body>

<div class="container">
    <!-- 登入與頻道選擇畫面 -->
    <div id="login-screen">
        <h2>對講機 Web App</h2>
        <input type="text" id="username" placeholder="輸入您的暱稱" required>
        <input type="text" id="room-id" placeholder="輸入頻道名稱 (如: 101)" required>
        <button class="btn-join" onclick="joinRoom()">進入頻道</button>
    </div>

    <!-- 對講機主畫面 -->
    <div id="ptt-screen">
        <div class="room-title">目前頻道: <span id="display-room"></span></div>
        
        <!-- 按住說話按鈕 -->
        <div id="ptt-btn">按住說話</div>
        
        <div id="status-text">準備就緒，按住按鈕開始廣播</div>
    </div>
</div>

<script>
    // ==========================================
    // ⚠️ 請將下方替換為您自己的 Firebase 設定！
    // ==========================================
    const firebaseConfig = {
        apiKey: "YOUR_API_KEY",
        authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
        databaseURL: "https://YOUR_PROJECT_ID-default-rtdb.firebaseio.com",
        projectId: "YOUR_PROJECT_ID",
        storageBucket: "YOUR_PROJECT_ID.appspot.com",
        messagingSenderId: "YOUR_SENDER_ID",
        appId: "YOUR_APP_ID"
    };
    
    // 初始化 Firebase
    if (!firebase.apps.length) {
        firebase.initializeApp(firebaseConfig);
    }
    const db = firebase.database();

    // 變數宣告
    let currentRoom = "";
    let userName = "";
    let mediaRecorder;
    let audioChunks = [];
    let isRecording = false;

    // 進入頻道
    async function joinRoom() {
        userName = document.getElementById('username').value.trim();
        currentRoom = document.getElementById('room-id').value.trim();

        if (!userName || !currentRoom) {
            alert('請輸入暱稱與頻道名稱！');
            return;
        }

        try {
            // 要求麥克風權限 (這是必須的)
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            setupMediaRecorder(stream);
            
            // 切換 UI
            document.getElementById('login-screen').style.display = 'none';
            document.getElementById('ptt-screen').style.display = 'flex';
            document.getElementById('display-room').innerText = currentRoom;

            // 監聽該頻道的 Firebase 資料變化 (接收他人的語音)
            listenToRoom();
            
        } catch (err) {
            alert('無法存取麥克風，請確保您已授權麥克風權限！');
            console.error(err);
        }
    }

    // 設定錄音機
    function setupMediaRecorder(stream) {
        mediaRecorder = new MediaRecorder(stream);

        mediaRecorder.ondataavailable = event => {
            if (event.data.size > 0) {
                audioChunks.push(event.data);
            }
        };

        mediaRecorder.onstop = async () => {
            const audioBlob = new Blob(audioChunks, { type: 'audio/webm' });
            audioChunks = []; // 清空緩存
            
            // 將錄音檔轉成 Base64 格式，方便存入 Firebase
            const reader = new FileReader();
            reader.readAsDataURL(audioBlob);
            reader.onloadend = () => {
                const base64Audio = reader.result;
                // 將語音資料推送到 Firebase 指定頻道
                db.ref('rooms/' + currentRoom).set({
                    sender: userName,
                    audioData: base64Audio,
                    timestamp: Date.now()
                });
            };
        };
    }

    // 綁定按鈕事件 (支援手機觸控與電腦滑鼠)
    const pttBtn = document.getElementById('ptt-btn');
    const statusText = document.getElementById('status-text');

    // 按下按鈕：開始錄音
    function startTalking(e) {
        e.preventDefault(); // 防止手機雙點擊放大
        if (isRecording) return;
        isRecording = true;
        
        pttBtn.classList.add('speaking');
        pttBtn.innerText = "錄音中...";
        statusText.innerText = "您正在廣播...";
        
        audioChunks = [];
        mediaRecorder.start();
    }

    // 放開按鈕：停止錄音並發送
    function stopTalking(e) {
        e.preventDefault();
        if (!isRecording) return;
        isRecording = false;

        pttBtn.classList.remove('speaking');
        pttBtn.innerText = "按住說話";
        statusText.innerText = "語音發送中...";
        
        mediaRecorder.stop();
        setTimeout(() => {
            statusText.innerText = "準備就緒，按住按鈕開始廣播";
        }, 1000);
    }

    // 手機觸控事件
    pttBtn.addEventListener('touchstart', startTalking);
    pttBtn.addEventListener('touchend', stopTalking);
    pttBtn.addEventListener('touchcancel', stopTalking);
    
    // 電腦滑鼠事件
    pttBtn.addEventListener('mousedown', startTalking);
    pttBtn.addEventListener('mouseup', stopTalking);
    pttBtn.addEventListener('mouseleave', stopTalking);

    // 監聽頻道中的新語音
    let lastTimestamp = Date.now(); // 忽略進入頻道前的舊歷史訊息
    function listenToRoom() {
        db.ref('rooms/' + currentRoom).on('value', (snapshot) => {
            const data = snapshot.val();
            if (data && data.timestamp > lastTimestamp && data.sender !== userName) {
                // 有新的語音，且不是自己發的
                lastTimestamp = data.timestamp;
                statusText.innerText = `🔊 ${data.sender} 正在說話...`;
                
                // 播放語音
                const audio = new Audio(data.audioData);
                audio.play();
                
                audio.onended = () => {
                    statusText.innerText = "準備就緒，按住按鈕開始廣播";
                };
            }
        });
    }
</script>

</body>
</html>
