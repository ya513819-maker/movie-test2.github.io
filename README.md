<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>敦克爾克：抉擇時刻 - 互動學習</title>
    <!-- 載入 Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        'dunkirk-blue': '#1a202c',
                        'sand-color': '#e5e5e0',
                        'accent-orange': '#f97316',
                    },
                    fontFamily: {
                        sans: ['Inter', 'Noto Sans TC', 'sans-serif'],
                    }
                }
            }
        }
    </script>
    <style>
        body {
            background-color: #f7f7f5;
            min-height: 100vh;
        }
        .card-shadow {
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -2px rgba(0, 0, 0, 0.06);
        }
        .person-card {cursor: grab;transition: transform 0.2s;}
        .person-card:active {cursor: grabbing;}
        .dragging {opacity: 0.7;transform: scale(1.05);z-index: 50;}
        .drop-zone {min-height: 12rem;border: 3px dashed #9ca3af;}
        .drop-zone.drag-over {border-color: #f97316;background-color: #fef3c7;}
    </style>
</head>
<body class="font-sans text-dunkirk-blue">

    <div id="app" class="max-w-4xl mx-auto p-4 md:p-8">
        <!-- Header -->
        <header class="text-center mb-8 border-b-2 border-accent-orange pb-4">
            <h1 class="text-3xl md:text-4xl font-extrabold text-dunkirk-blue">《敦克爾克：抉擇時刻》</h1>
            <p class="text-lg text-gray-600 mt-2">從絕望沙灘到人性光輝</p>
            <p id="user-info" class="text-sm mt-2 text-gray-500">連線中...</p>
        </header>

        <!-- Dynamic Content Area -->
        <div id="content">
            <div class="text-center p-8">
                <h2 class="text-2xl font-bold mb-4 text-dunkirk-blue">正在準備個人模擬...</h2>
                <p class="text-lg text-gray-600">連線至資料庫中，請稍候。</p>
                <svg class="animate-spin h-8 w-8 text-dunkirk-blue mx-auto mt-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
            </div>
        </div>
    </div>

    <!-- Firebase SDKs & 主程式 -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, arrayUnion } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Firebase設定直接用明文，勿用typeof，Pages才會生效！
        const firebaseConfig = {
            apiKey: "AIzaSyAk787OaW5KBN0jWbyRMhJxwlYWqMLUx5k",
            authDomain: "test-dd39e.firebaseapp.com",
            projectId: "test-dd39e",
            storageBucket: "test-dd39e.firebasestorage.app",
            messagingSenderId: "744196429035",
            appId: "1:744196429035:web:c262533b33e8d390359aae",
            measurementId: "G-32FRBE0RL4"
        };

        // 初始化Firebase
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);

        // 自動匿名登入並初始化
        let userId = null;
        let userChoicesRef = null;
        let userName = null;
        let userClass = null;
        let currentPhase = 'loading';
        let socraticCritique = null;
        let currentUserId = 'unknown';

        // 匿名登入後帶入資料流程
        onAuthStateChanged(auth, async (user) => {
            if (!user) await signInAnonymously(auth); //匿名登入
            userId = auth.currentUser.uid;
            userChoicesRef = doc(db, `artifacts/default-dunkirk-app/users/${userId}/dunkirk_data/choices`);
            //可繼續執行init & 畫面流程
            document.getElementById('user-info').textContent = "個人身份驗證成功";
            // ... 你原來initFirebase、startApp等資料流程可直接跟著觸發
        });

        // 以下直接接你的canvas互動/遊戲記錄/UI流程原始碼，JS功能與原本完全一致
        // 例如你所有 phase1、phase2、registration 畫面切換、資料存/讀邏輯直接貼在這個script內

    </script>
</body>
</html>
