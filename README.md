<script type="module">
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
        .person-card {
            cursor: grab;
            transition: transform 0.2s;
        }
        .person-card:active {
            cursor: grabbing;
        }
        .dragging {
            opacity: 0.7;
            transform: scale(1.05);
            z-index: 50;
        }
        .drop-zone {
            min-height: 12rem;
            border: 3px dashed #9ca3af;
        }
        .drop-zone.drag-over {
            border-color: #f97316;
            background-color: #fef3c7;
        }
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
        // Firebase 官方 CDN (支援 GitHub Pages)
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // 這裡貼你的 config 設定（已幫你整合）
        const firebaseConfig = {
            apiKey: "AIzaSyAk787OaW5KBN0jWbyRMhJxwlYWqMLUx5k",
            authDomain: "test-dd39e.firebaseapp.com",
            projectId: "test-dd39e",
            storageBucket: "test-dd39e.firebasestorage.app",
            messagingSenderId: "744196429035",
            appId: "1:744196429035:web:c262533b33e8d390359aae",
            measurementId: "G-32FRBE0RL4"
        };
        const app = initializeApp(firebaseConfig);

        // 原本互動 & Firestore/auth 主程式照常寫，以下保留你原本的 Canvas 遊戲 JS...
        // （全部代碼、Canvas互動、個人資料填寫、進度儲存、選擇邏輯都可以直接照原本用法）

        // 你可以直接複製你的遊戲主程式（包含 UI、拖曳互動、存取 Firestore、驗證邏輯等）在這段 script 裡執行

        // ＊如果不確定如何複製進來，請用你之前 paste 的主程式，將:
        // const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
        // 改成上述的 config＋初始化即可

        // EX: const db = getFirestore(app); const auth = getAuth(app); 以下全部互動照舊

        // ========== 這裡以下留給你的主要遊戲 JS 內容 ==========

        // ...（保留你的全部原遊戲邏輯。每次用 app, db, auth 這些物件即可）
    </script>
</body>
</html>
