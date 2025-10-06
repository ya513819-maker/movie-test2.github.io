<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>敦克爾克大行動：抉擇時刻 - 互動學習</title>

  <!-- Tailwind（CDN 用於教學/DEMO，可忽略警告） -->
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
          },
        },
      },
    };
  </script>

  <style>
    body { background-color: #f7f7f5; min-height: 100vh; }
    .card-shadow { box-shadow: 0 4px 6px -1px rgba(0,0,0,.1), 0 2px 4px -2px rgba(0,0,0,.06); }
    .person-card { cursor: grab; transition: transform .2s; }
    .person-card:active { cursor: grabbing; }
    .dragging { opacity:.7; transform: scale(1.05); z-index: 50; }
    .drop-zone { min-height: 12rem; border: 3px dashed #9ca3af; }
    .drop-zone.drag-over { border-color:#f97316; background-color:#fef3c7; }
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

    <!-- Dynamic Content -->
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

  <!-- Firebase + 主程式 -->
  <script type="module">
    // Firebase CDN
    import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
    import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
    import { getFirestore, doc, getDoc, setDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

    // 你的 Firebase 設定
    const firebaseConfig = {
      apiKey: "AIzaSyAk787OaW5KBN0jWbyRMhJxwlYWqMLUx5k",
      authDomain: "test-dd39e.firebaseapp.com",
      projectId: "test-dd39e",
      storageBucket: "test-dd39e.firebasestorage.app",
      messagingSenderId: "744196429035",
      appId: "1:744196429035:web:c262533b33e8d390359aae",
      measurementId: "G-32FRBE0RL4"
    };

    // 初始化
    const app = initializeApp(firebaseConfig);
    const db = getFirestore(app);
    const auth = getAuth(app);

    // LLM（Gemini）設定：使用你提供的 API Key
    const API_KEY = "AIzaSyDtH5v550LZk_mbTQGCwc8po3y70_HCL3Q";
    const GEMINI_API_MODEL = "gemini-2.5-flash-preview-05-20";
    const GEMINI_API_URL = `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_API_MODEL}:generateContent?key=${API_KEY}`;

    // 全域狀態
    const APP_ID = 'default-dunkirk-app';
    let userId = null;
    let userChoicesRef = null;
    let userName = null;
    let userClass = null;
    let currentPhase = 'loading';
    let socraticCritique = null;

    // 資料設定
    const PEOPLE_DATA = [
      { id: 'W01', type: 'WoundedSoldier', label: '重傷士兵', count: 5, status: '無法戰鬥', priority: 1, color: 'bg-red-200' },
      { id: 'Y01', type: 'YoungSoldier', label: '年輕士兵', count: 10, status: '可戰鬥', priority: 2, color: 'bg-green-200' },
      { id: 'C01', type: 'CivilianSailor', label: '航海平民', count: 5, status: '專業技能', priority: 3, color: 'bg-yellow-200' },
      { id: 'O01', type: 'Orphan', label: '年幼孤兒', count: 5, status: '最無助', priority: 4, color: 'bg-purple-200' },
      { id: 'N01', type: 'Nurse', label: '多語護士', count: 5, status: '醫療翻譯', priority: 5, color: 'bg-blue-200' },
    ];
    const MORAL_PRINCIPLES = {
      A: '效用最大化 (對未來戰爭有最大幫助)',
      B: '弱勢優先 (優先拯救最無助、最弱勢者)',
      C: '公平原則 (確保每類人都有被拯救的機會)',
      D: '長期價值 (具備關鍵技術或知識者優先)',
    };
    const SCENES = {
      hope: [
        { id: 1, description: '民船艦隊出現在地平線' },
        { id: 2, description: '最後一架噴火式戰機被燒毀' },
        { id: 3, description: '士兵在火車上受到老者歡迎' },
      ],
      despair: [
        { id: 4, description: '士兵被困在沉沒的船艙內' },
        { id: 5, description: '沙灘上排隊等待轟炸' },
        { id: 6, description: '士兵將同袍趕出超載的船' },
      ]
    };

    // 匿名登入與啟動流程
    onAuthStateChanged(auth, async (user) => {
      try {
        if (!user) await signInAnonymously(auth);
        userId = auth.currentUser.uid;
        userChoicesRef = doc(db, `artifacts/${APP_ID}/users/${userId}/dunkirk_data/choices`);
        try {
          const snap = await getDoc(userChoicesRef);
          const data = snap.data() || {};
          userName = data.userName || null;
          userClass = data.userClass || null;
        } catch (err) {
          console.warn('[init] 初次 getDoc 失敗（可忽略，文件待建立）：', err);
        }
        document.getElementById('user-info').textContent = '個人身份驗證成功';
        startApp();
      } catch (e) {
        console.error('[auth] 身分流程失敗：', e);
        document.getElementById('content').innerHTML =
          '<p class="text-red-600 text-center">錯誤：身分驗證流程失敗，請重整再試。</p>';
      }
    });

    async function checkUserProgress() {
      try {
        if (!userChoicesRef) return { isRegistered: false, phase1: false, phase2: false };
        const docSnap = await getDoc(userChoicesRef);
        const data = docSnap.data() || {};
        return {
          isRegistered: !!(data.userName && data.userClass),
          phase1: !!data.phase1Completed,
          phase2: !!data.phase2Completed,
        };
      } catch (e) {
        console.warn('[flow] checkUserProgress 讀取失敗，走註冊流程：', e);
        return { isRegistered: false, phase1: false, phase2: false };
      }
    }

    function startApp() {
      checkUserProgress().then(progress => {
        if (!progress.isRegistered) {
          currentPhase = 'registration';
          renderRegistrationScreen();
        } else if (progress.phase1 && progress.phase2) {
          currentPhase = 'results';
          renderResults();
        } else if (progress.phase1) {
          currentPhase = 'phase2';
          renderPhase2();
        } else {
          currentPhase = 'start';
          renderStartScreen();
        }
      }).catch(err => {
        console.error('[flow] startApp error:', err);
        document.getElementById('content').innerHTML =
          '<p class="text-red-600 text-center">載入流程發生錯誤，請重整再試。</p>';
      });
    }

    // UI 畫面
    function renderRegistrationScreen() {
      document.getElementById('content').innerHTML = `
        <div class="text-center p-8 bg-white rounded-xl card-shadow">
          <h2 class="text-3xl font-extrabold text-dunkirk-blue mb-4">開始個人模擬：請填寫基本資訊</h2>
          <p class="text-lg text-gray-700 mb-6">請輸入您的班級與姓名，以便儲存您的進度與結果。</p>
          <form id="registration-form" class="space-y-4 max-w-sm mx-auto">
            <input type="text" id="reg-class" placeholder="您的班級 (例如: 高二丙班)" required
              class="w-full p-3 border border-gray-300 rounded-lg focus:ring-accent-orange focus:border-accent-orange">
            <input type="text" id="reg-name" placeholder="您的姓名 (例如: 王小明)" required
              class="w-full p-3 border border-gray-300 rounded-lg focus:ring-accent-orange focus:border-accent-orange">
            <button type="submit"
              class="w-full bg-accent-orange text-white font-bold py-3 px-8 rounded-xl hover:bg-orange-700 transition-all duration-300 shadow-lg hover:shadow-xl text-lg tracking-wider">
              確認並進入歡迎畫面
            </button>
            <p id="reg-error" class="text-red-500 text-sm mt-3"></p>
          </form>
        </div>
      `;
      document.getElementById('registration-form').addEventListener('submit', submitRegistration);
    }

    async function submitRegistration(e) {
      e.preventDefault();
      const inputClass = document.getElementById('reg-class').value.trim();
      const inputName = document.getElementById('reg-name').value.trim();
      if (!inputClass || !inputName) {
        document.getElementById('reg-error').textContent = '班級與姓名皆為必填。';
        return;
      }
      try {
        userClass = inputClass;
        userName = inputName;
        await setDoc(userChoicesRef, {
          userName, userClass, isRegistered: true
        }, { merge: true });
        document.getElementById('user-info').textContent = `玩家: ${userClass} / ${userName}`;
        renderStartScreen();
      } catch (err) {
        console.error('[reg] 儲存失敗：', err);
        document.getElementById('reg-error').textContent = '儲存資訊失敗，請重試。';
      }
    }

    function renderStartScreen() {
      document.getElementById('content').innerHTML = `
        <div class="text-center p-8 bg-white rounded-xl card-shadow">
          <h2 class="text-3xl font-extrabold text-dunkirk-blue mb-4">歡迎來到敦克爾克沙灘</h2>
          <p class="text-lg text-gray-700 mb-6">玩家：<span class="font-bold">${userClass || '-'} / ${userName || '-'}</span></p>
          <p class="text-lg text-gray-700 mb-6">這是一個道德與人性的抉擇模擬。你將扮演一位小型救援船船長，面臨時間和資源的極限挑戰。</p>
          <button id="start-dilemma-btn"
            class="bg-accent-orange text-white font-bold py-4 px-8 rounded-xl hover:bg-orange-700 transition-all duration-300 shadow-lg hover:shadow-xl transform hover:scale-105 text-xl tracking-wider">
            點擊開始：進入船長困境
          </button>
          <p class="text-sm text-gray-500 mt-6">（注意：一旦開始，第一階段的限時挑戰將立即啟動。）</p>
        </div>
      `;
      document.getElementById('start-dilemma-btn').addEventListener('click', renderPhase1Dilemma);
    }

    // Phase 1
    let savedCount = 0;
    let selectedPeople = {};
    let timerInterval;

    function renderPhase1Dilemma() {
      savedCount = 0;
      selectedPeople = {};

      const peopleHTML = PEOPLE_DATA.map(group =>
        Array(group.count).fill(0).map((_, i) =>
          `<div id="${group.id}-${i+1}" data-type="${group.type}" data-label="${group.label}"
             class="person-card card-shadow bg-sand-color p-3 m-1 rounded-lg text-sm text-center select-none" draggable="true">
             ${group.label} #${i + 1}<br><span class="text-xs text-gray-500">${group.status}</span>
           </div>`
        ).join('')
      ).join('');

      document.getElementById('content').innerHTML = `
        <h2 class="text-2xl font-bold mb-4 text-dunkirk-blue">第一階段：船長困境（道德困境沙盤推演）</h2>
        <div class="p-4 rounded-xl bg-yellow-100/50 mb-4 text-sm md:text-base">
          <p class="font-bold text-lg text-accent-orange mb-2">情境：你的船艙只能再搭載 <span id="capacity-display" class="text-2xl font-extrabold">10</span> 人。</p>
          <p>請在 3 分鐘內，從下方的 30 位等待者中，拖曳 10 位進入「船艙區域」。時間結束後，系統將自動記錄你的人選。</p>
        </div>
        <div class="text-center mb-4">
          <span id="timer" class="text-3xl font-extrabold text-red-600 p-2 rounded-lg bg-red-100 card-shadow">03:00</span>
        </div>
        <div id="drop-zone" class="drop-zone rounded-xl p-3 mb-6 flex flex-wrap justify-center items-center bg-gray-100">
          <p class="text-gray-500 text-lg">拖曳 10 位待救者到此區域 (船艙)</p>
        </div>
        <div class="flex justify-between items-center mb-4">
          <h3 class="text-xl font-semibold">待救者 (沙灘上的 30 人)</h3>
          <button id="submit-p1" class="px-4 py-2 bg-accent-orange text-white rounded-lg font-semibold hover:bg-orange-700 transition-colors hidden">提交選擇</button>
        </div>
        <div id="people-container" class="grid grid-cols-5 md:grid-cols-6 lg:grid-cols-8 gap-2 overflow-y-auto max-h-[40vh] p-2 bg-white rounded-xl card-shadow">
          ${peopleHTML}
        </div>
      `;
      setupDragDropListeners();
      startTimer(180);
    }

    function setupDragDropListeners() {
      const dropZone = document.getElementById('drop-zone');
      const peopleContainer = document.getElementById('people-container');
      const cards = peopleContainer.querySelectorAll('.person-card');
      const submitButton = document.getElementById('submit-p1');

      cards.forEach(card => {
        card.addEventListener('dragstart', (e) => {
          e.dataTransfer.setData('text/plain', e.target.id);
          e.target.classList.add('dragging');
        });
        card.addEventListener('dragend', (e) => e.target.classList.remove('dragging'));
        card.addEventListener('click', (e) => {
          if (e.currentTarget.parentElement.id === 'people-container' && savedCount < 10) {
            moveCard(e.currentTarget, dropZone);
          } else if (e.currentTarget.parentElement.id === 'drop-zone') {
            moveCard(e.currentTarget, peopleContainer);
          }
        });
      });

      dropZone.addEventListener('dragover', (e) => { e.preventDefault(); dropZone.classList.add('drag-over'); });
      dropZone.addEventListener('dragleave', () => dropZone.classList.remove('drag-over'));
      dropZone.addEventListener('drop', (e) => {
        e.preventDefault();
        dropZone.classList.remove('drag-over');
        const id = e.dataTransfer.getData('text/plain');
        const card = document.getElementById(id);
        if (card && card.parentElement.id === 'people-container' && savedCount < 10) {
          moveCard(card, dropZone);
        }
      });

      submitButton.addEventListener('click', submitPhase1);

      function updateCapacityDisplay() {
        document.getElementById('capacity-display').textContent = 10 - savedCount;
        if (savedCount === 10) {
          if (dropZone.children.length === 0) dropZone.innerHTML = '';
          submitButton.classList.remove('hidden');
        } else {
          if (dropZone.children.length === 0 || dropZone.children[0].tagName !== 'DIV') {
            dropZone.innerHTML = '<p class="text-gray-500 text-lg">拖曳 10 位待救者到此區域 (船艙)</p>';
          }
          submitButton.classList.add('hidden');
        }
      }

      function moveCard(card, destination) {
        if (destination.id === 'drop-zone') {
          savedCount++;
          selectedPeople[card.id] = { type: card.dataset.type, label: card.dataset.label };
        } else {
          savedCount--;
          delete selectedPeople[card.id];
        }
        destination.appendChild(card);
        updateCapacityDisplay();
      }
    }

    function startTimer(duration) {
      let time = duration;
      const timerDisplay = document.getElementById('timer');
      clearInterval(timerInterval);
      timerInterval = setInterval(() => {
        const minutes = String(Math.floor(time / 60)).padStart(2, '0');
        const seconds = String(time % 60).padStart(2, '0');
        timerDisplay.textContent = `${minutes}:${seconds}`;
        if (--time < 0) {
          clearInterval(timerInterval);
          timerDisplay.textContent = '時間到！';
          submitPhase1(true);
        }
      }, 1000);
    }

    function submitPhase1(isForced = false) {
      clearInterval(timerInterval);
      if (!isForced && savedCount === 0) {
        if (!confirm("您還沒有拯救任何人。確定要以空船提交嗎？")) {
          startTimer(1);
          return;
        }
      }
      const selection = Object.values(selectedPeople).reduce((acc, p) => {
        acc[p.type] = (acc[p.type] || 0) + 1;
        return acc;
      }, {});
      renderMoralPrincipleForm(selection);
    }

    function renderMoralPrincipleForm(selection) {
      const moralPrinciplesHTML = Object.entries(MORAL_PRINCIPLES).map(([k, v]) => `
        <label class="block mb-2 p-3 bg-white rounded-lg card-shadow hover:bg-gray-50 transition-colors cursor-pointer">
          <input type="checkbox" name="moral_principle" value="${k}" class="mr-3 accent-accent-orange"> ${v}
        </label>
      `).join('');
      const selectionDisplay = Object.entries(selection).map(([type, count]) => {
        const label = PEOPLE_DATA.find(p => p.type === type)?.label || '未知';
        return `<span class="inline-block bg-dunkirk-blue text-white text-xs px-3 py-1 m-1 rounded-full">${label} x ${count}</span>`;
      }).join('') || '<span class="text-gray-500">無人被拯救。</span>';

      document.getElementById('content').innerHTML = `
        <h2 class="text-2xl font-bold mb-4 text-dunkirk-blue">第二步驟：選擇您的道德原則</h2>
        <div class="p-4 rounded-xl bg-yellow-100/50 mb-6 text-sm md:text-base">
          <p class="font-bold text-lg text-accent-orange mb-2">你的選擇：</p>
          ${selectionDisplay}
          <p class="mt-4">請選擇 1 到 3 個最能代表你決策依據的道德原則。</p>
        </div>
        <form id="moral-form" class="space-y-3">
          ${moralPrinciplesHTML}
          <button type="submit" class="w-full mt-6 px-4 py-3 bg-accent-orange text-white rounded-lg font-semibold text-lg hover:bg-orange-700 transition-colors">確認原則並進入下一階段</button>
          <p id="error-message" class="text-red-500 text-center"></p>
        </form>
      `;
      document.getElementById('moral-form').addEventListener('submit', async (e) => {
        e.preventDefault();
        const selected = Array.from(e.target.querySelectorAll('input[name="moral_principle"]:checked')).map(el => el.value);
        if (selected.length === 0 || selected.length > 3) {
          document.getElementById('error-message').textContent = '請選擇 1 到 3 個道德原則！';
          return;
        }
        try {
          const savedPeopleCount = Object.entries(selection).reduce((acc, [type, count]) => {
            if (count > 0) acc[type] = count;
            return acc;
          }, {});
          await setDoc(userChoicesRef, {
            userId, phase1Completed: true, timestamp: new Date().toISOString(),
            savedCount, savedPeople: savedPeopleCount, moralPrinciples: selected
          }, { merge: true });
          renderPhase2();
        } catch (err) {
          console.error('[p1] 儲存失敗：', err);
          document.getElementById('error-message').textContent = '儲存失敗，請檢查連線。';
        }
      });
    }

    function renderPhase2() {
      const hopeSceneHTML = SCENES.hope.map(s => `
        <label class="flex items-center p-3 bg-white rounded-lg card-shadow hover:bg-gray-50 transition-colors cursor-pointer">
          <input type="radio" name="hope_vote" value="${s.id}" data-desc="${s.description}" class="mr-3 accent-dunkirk-blue" required>
          ${s.description}
        </label>
      `).join('');
      const despairSceneHTML = SCENES.despair.map(s => `
        <label class="flex items-center p-3 bg-white rounded-lg card-shadow hover:bg-gray-50 transition-colors cursor-pointer">
          <input type="radio" name="despair_vote" value="${s.id}" data-desc="${s.description}" class="mr-3 accent-dunkirk-blue" required>
          ${s.description}
        </label>
      `).join('');

      document.getElementById('content').innerHTML = `
        <h2 class="text-2xl font-bold mb-4 text-dunkirk-blue">第二階段：無名英雄與電影反思（深度互動）</h2>
        <p class="mb-6 text-gray-600">請完成以下所有任務，將「敦克爾克精神」延伸到您的日常觀察，並進行深度道德反思。</p>
        <form id="phase2-form" class="space-y-6">
          <div class="p-4 bg-sand-color rounded-xl card-shadow">
            <h3 class="text-xl font-semibold mb-3">任務一：現代的敦克爾克英雄 (個人反思)</h3>
            <input type="text" id="hero-identity" placeholder="例如: 清潔隊員" required class="w-full p-2 border border-gray-300 rounded-lg mb-2">
            <input type="text" id="hero-tag" placeholder="例如: #默默堅守" required class="w-full p-2 border border-gray-300 rounded-lg">
          </div>
          <div class="p-4 bg-white rounded-xl card-shadow">
            <h3 class="text-xl font-semibold mb-3 text-green-700">任務二：希望時刻投票與理由</h3>
            <div class="space-y-2">${hopeSceneHTML}</div>
            <div class="mt-4">
              <label class="block text-sm font-medium mb-1">請簡述理由 (50字內):</label>
              <textarea id="hope-reason" rows="2" required class="w-full p-2 border border-gray-300 rounded-lg resize-none"></textarea>
            </div>
          </div>
          <div class="p-4 bg-white rounded-xl card-shadow">
            <h3 class="text-xl font-semibold mb-3 text-red-700">任務三：絕望時刻投票與理由</h3>
            <div class="space-y-2">${despairSceneHTML}</div>
            <div class="mt-4">
              <label class="block text-sm font-medium mb-1">請簡述理由 (50字內):</label>
              <textarea id="despair-reason" rows="2" required class="w-full p-2 border border-gray-300 rounded-lg resize-none"></textarea>
            </div>
          </div>
          <div class="p-4 bg-yellow-100/50 rounded-xl card-shadow">
            <h3 class="text-xl font-semibold mb-3 text-dunkirk-blue">任務四：道德反思與矛盾 (魔鬼代言人挑戰)</h3>
            <textarea id="ethical-reflection" rows="4" required class="w-full p-2 border border-gray-300 rounded-lg resize-none" placeholder="請寫下你的反思，這是課程討論的重點。"></textarea>
          </div>
          <button type="submit" class="w-full px-4 py-3 bg-dunkirk-blue text-white rounded-lg font-semibold text-lg hover:bg-gray-700 transition-colors">提交反思結果並查看個人分析</button>
          <p id="error-message-p2" class="text-red-500 text-center"></p>
        </form>
      `;
      document.getElementById('phase2-form').addEventListener('submit', savePhase2Data);
    }

    // 強化版 Gemini 呼叫：多路擷取與保底
    async function fetchWithExponentialBackoff(url, options, maxRetries = 5) {
      for (let i = 0; i < maxRetries; i++) {
        try {
          const response = await fetch(url, options);
          if (response.ok) return response;
          if (response.status !== 429 && response.status < 500) {
            throw new Error(`API status ${response.status}: ${await response.text()}`);
          }
        } catch (error) {
          if (i === maxRetries - 1) throw error;
        }
        const delay = Math.pow(2, i) * 1000 + Math.random() * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
      throw new Error("Max retries exceeded.");
    }

    async function generateSocraticCritique(data) {
      const systemPrompt =
        "你就是蘇格拉底的靈魂。你的目標不是給予答案，而是根據學生提供的倫理和道德數據，提出一個深入、挑戰他們核心假設、偏見或矛盾的批判或反問。你的回應必須以繁體中文呈現，以一個有力的詰問開頭，並力求簡潔（不超過 150 字）。";

      const userQuery = `請針對以下學生的「敦克爾克：抉擇時刻」第二階段提交數據，給出一個蘇格拉底式的質疑：

1. 現代無名英雄身份/標籤: ${data.heroIdentity} / ${data.heroTag}
2. 最能代表「希望」的場景: ${data.hopeSceneDescription}
3. 選擇希望的理由: ${data.hopeReason}
4. 最能代表「絕望」的場景: ${data.despairSceneDescription}
5. 選擇絕望的理由: ${data.despairReason}
6. 道德反思與矛盾: ${data.ethicalReflection}

質疑目標：挑戰他們的英雄定義或道德反思的深度。`;

      const payload = {
        contents: [{ parts: [{ text: userQuery }] }],
        systemInstruction: { parts: [{ text: systemPrompt }] }
      };

      try {
        const response = await fetchWithExponentialBackoff(
          GEMINI_API_URL,
          { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) }
        );
        const result = await response.json();

        // 多路擷取：盡可能從不同欄位拿文字
        const candidates = result?.candidates || [];
        let text = "";

        // 1) 官方常見路徑
        if (!text && candidates[0]?.content?.parts?.length) {
          const firstPart = candidates[0].content.parts.find(p => typeof p.text === 'string' && p.text.trim().length > 0);
          text = firstPart?.text || "";
        }
        // 2) 有時模型把文字放在「candidates[0].content」直接為字串（較少見）
        if (!text && typeof candidates[0]?.content === 'string') {
          text = candidates[0].content;
        }
        // 3) 嘗試把所有 parts 的 text 串接（防止分段）
        if (!text && candidates[0]?.content?.parts?.length) {
          text = candidates[0].content.parts.map(p => p.text).filter(Boolean).join('\n').trim();
        }
        // 4) 退路：看有沒有 promptFeedback 或其他欄位（少見）
        if (!text && result?.promptFeedback?.blockReason) {
          text = `（內容被系統過濾：${result.promptFeedback.blockReason}）請嘗試更具體的描述。`;
        }

        // 最終保底
        if (!text || !text.trim()) {
          text = "你說「電影很好看」，但這是審美評價而非倫理分析：若把欣賞與道德判斷混為一談，你是否忽略了「行動後果」與「動機」的差異？";
        }
        return text.trim();
      } catch (error) {
        console.error("Gemini API call failed:", error);
        return "（目前無法取得 AI 詰問，請先思考：你的理由若被相反立場質疑，最脆弱的環節是什麼？）";
      }
    }

    async function savePhase2Data(e) {
      e.preventDefault();
      try {
        const heroIdentity = document.getElementById('hero-identity').value.trim();
        const heroTag = document.getElementById('hero-tag').value.trim();
        const hopeChecked = document.querySelector('input[name="hope_vote"]:checked');
        const despairChecked = document.querySelector('input[name="despair_vote"]:checked');
        if (!heroIdentity || !heroTag || !hopeChecked || !despairChecked) {
          document.getElementById('error-message-p2').textContent = '請完成所有欄位填寫！';
          return;
        }
        const phase2Data = {
          phase2Completed: true,
          heroIdentity,
          heroTag,
          hopeSceneId: parseInt(hopeChecked.value),
          hopeSceneDescription: hopeChecked.dataset.desc,
          hopeReason: document.getElementById('hope-reason').value.trim(),
          despairSceneId: parseInt(despairChecked.value),
          despairSceneDescription: despairChecked.dataset.desc,
          despairReason: document.getElementById('despair-reason').value.trim(),
          ethicalReflection: document.getElementById('ethical-reflection').value.trim(),
        };

        // 先顯示等待
        document.getElementById('content').innerHTML = `
          <div class="text-center p-8 bg-white rounded-xl card-shadow">
            <h2 class="text-2xl font-bold mb-4 text-accent-orange">正在等待蘇格拉底的回覆...</h2>
            <p class="text-lg text-gray-600">這可能需要幾秒鐘，請不要關閉頁面。</p>
            <svg class="animate-spin h-8 w-8 text-dunkirk-blue mx-auto mt-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
              <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
              <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
          </div>
        `;

        // 產生 AI 詰問
        socraticCritique = await generateSocraticCritique(phase2Data);

        // 儲存
        await setDoc(userChoicesRef, {
          ...phase2Data,
          socraticCritique,
          timestampP2: new Date().toISOString(),
        }, { merge: true });

        renderResults();
      } catch (e2) {
        console.error('[p2] 提交或生成失敗：', e2);
        document.getElementById('error-message-p2').textContent = '提交失敗，請檢查網路連線或重試。';
        renderResults();
      }
    }

    function renderResults() {
      document.getElementById('content').innerHTML = `
        <div class="text-center p-8">
          <h2 class="text-2xl font-bold mb-4 text-dunkirk-blue">正在分析你的抉擇...</h2>
          <svg class="animate-spin h-8 w-8 text-dunkirk-blue mx-auto mt-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
        </div>
      `;

      getDoc(userChoicesRef).then(docSnap => {
        const userData = docSnap.data() || {};
        const p1Summary = Object.entries(userData.savedPeople || {}).map(([type, count]) => {
          const label = PEOPLE_DATA.find(p => p.type === type)?.label || '未知';
          return `<span class="inline-block bg-dunkirk-blue text-white text-sm px-3 py-1 m-1 rounded-full">${label} x ${count}</span>`;
        }).join('') || '<p class="text-gray-500">無人被拯救。</p>';

        const principleSummary = (userData.moralPrinciples || []).map(k => MORAL_PRINCIPLES[k]).join('；') || '未選擇原則';
        const heroSummary = `${userData.heroIdentity || '無'} (${userData.heroTag || '無'})`;
        const hopeDesc = userData.hopeSceneDescription || '無';
        const despairDesc = userData.despairSceneDescription || '無';
        
        // 強化版 critique 取值：保底有內容且不顯示「未載入」
        const critique = (userData.socraticCritique || socraticCritique || '').trim()
          || '你說「電影很好看」，但這是審美評價而非倫理分析：若把欣賞與道德判斷混為一談，你是否忽略了「行動後果」與「動機」的差異？';

        document.getElementById('content').innerHTML = `
          <h2 class="text-2xl font-bold mb-6 text-dunkirk-blue border-b-2 border-accent-orange pb-2">個人分析：你的抉擇總結</h2>
          <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
            <div class="lg:col-span-2 space-y-6">
              <div class="p-6 bg-gray-50 rounded-xl card-shadow border-l-4 border-accent-orange">
                <h3 class="text-xl font-bold mb-3 text-dunkirk-blue">你的船長日誌 (抉擇回顧)</h3>
                <p class="text-lg font-semibold mb-2">拯救人數: ${userData.savedCount || 0} 人</p>
                <p class="text-sm text-gray-700 mb-4">人選: ${p1Summary}</p>
                <p class="text-lg font-semibold mb-2">決策原則: </p>
                <p class="text-sm text-gray-700">${principleSummary}</p>
              </div>
              <div class="p-6 bg-white rounded-xl card-shadow border-l-4 border-dunkirk-blue">
                <h3 class="text-xl font-bold mb-3 text-dunkirk-blue">你的道德反思</h3>
                <div class="text-sm space-y-2 text-gray-700">
                  <p><strong>無名英雄：</strong> ${heroSummary}</p>
                  <p><strong>希望場景 (${hopeDesc}) 理由：</strong> ${userData.hopeReason || '無'}</p>
                  <p><strong>絕望場景 (${despairDesc}) 理由：</strong> ${userData.despairReason || '無'}</p>
                  <p class="mt-4 border-t pt-2"><strong>倫理矛盾反思：</strong> ${userData.ethicalReflection || '無'}</p>
                </div>
              </div>
            </div>
            <div class="lg:col-span-1 p-6 bg-yellow-100 rounded-xl card-shadow border-t-8 border-yellow-500">
              <h3 class="text-xl font-bold mb-3 text-yellow-800">蘇格拉底的詰問</h3>
              <div class="text-base text-yellow-900 italic font-serif">
                <p>${critique}</p>
              </div>
            </div>
          </div>
        `;
      }).catch(e => {
        console.error('[results] 讀取失敗：', e);
        document.getElementById('content').innerHTML =
          '<p class="text-red-600 text-center">無法載入你的抉擇數據。</p>';
      });
    }

    // 安全防呆：10秒仍未進入主畫面，提示重整
    setTimeout(() => {
      if (currentPhase === 'loading') {
        document.getElementById('content').insertAdjacentHTML('beforeend',
          '<p class="text-center text-sm text-gray-500 mt-4">連線較久，請稍候或重整頁面。</p>');
      }
    }, 10000);
  </script>
</body>
</html>
