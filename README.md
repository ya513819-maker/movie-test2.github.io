    <script type="module">
        // Import the functions you need from the SDKs you need
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAnalytics } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-analytics.js"; // Added Analytics import
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Your web app's Firebase configuration - NOW USING YOUR PROVIDED CONFIG!
        const firebaseConfig = {
          apiKey: "AIzaSyAk787OaW5KBN0jWbyRMhJxwlYWqMLUx5k",
          authDomain: "test-dd39e.firebaseapp.com",
          projectId: "test-dd39e",
          storageBucket: "test-dd39e.firebasestorage.app",
          messagingSenderId: "744196429035",
          appId: "1:744196429035:web:c262533b33e8d390359aae",
          measurementId: "G-32FRBE0RL4"
        };

        let app;
        let db;
        let auth;
        let userId = null;
        let userChoicesRef = null;
        let analytics; // Declare analytics variable

        let userName = null;
        let userClass = null;
        let currentPhase = 'loading'; // loading, registration, start, phase1, phase2, results
        let currentUserId = 'unknown';
        let socraticCritique = null;

        const API_KEY = ""; // 請填寫你的 Gemini API Key
        const GEMINI_API_MODEL = "gemini-2.5-flash-preview-05-20";
        const GEMINI_API_URL = `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_API_MODEL}:generateContent?key=${API_KEY}`;


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

        function updateUserInfoDisplay() {
            let infoText = `個人身份驗證成功`;
            if (userName && userClass) {
                infoText = `玩家: ${userClass} / ${userName}`;
            }
            document.getElementById('user-info').textContent = infoText;
        }

        async function initFirebase() {
            try {
                app = initializeApp(firebaseConfig);
                analytics = getAnalytics(app); // Initialize Google Analytics here
                db = getFirestore(app);
                auth = getAuth(app);

                document.getElementById('user-info').textContent = '正在驗證身分...';

                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        currentUserId = user.uid;
                        userId = currentUserId;
                        userChoicesRef = doc(db, `artifacts/${firebaseConfig.appId}/users/${userId}/dunkirk_data/choices`);

                        const docSnap = await getDoc(userChoicesRef);
                        const userData = docSnap.data() || {};
                        userName = userData.userName || null;
                        userClass = userData.userClass || null;

                        updateUserInfoDisplay();
                        startApp();
                    } else {
                        try {
                            await signInAnonymously(auth);
                        } catch (anonError) {
                            console.error("Anonymous sign-in failed:", anonError);
                            document.getElementById('content').innerHTML = `<p class="text-red-600 text-center">致命錯誤：匿名登入失敗，請重試。</p>`;
                        }
                    }
                });

            } catch (error) {
                console.error("Firebase initialization failed:", error);
                document.getElementById('content').innerHTML = `<p class="text-red-600 text-center">錯誤：Firebase 連線或初始化失敗。詳情請看控制台。</p>`;
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
                    getDoc(userChoicesRef).then(docSnap => {
                        socraticCritique = docSnap.data()?.socraticCritique || null;
                        renderPhase2();
                    });
                } else {
                    currentPhase = 'start';
                    renderStartScreen();
                }
            });
        }

        async function checkUserProgress() {
            try {
                const docSnap = await getDoc(userChoicesRef);
                const data = docSnap.data() || {};
                return {
                    isRegistered: data.userName && data.userClass,
                    phase1: data.phase1Completed || false,
                    phase2: data.phase2Completed || false,
                };
            } catch (e) {
                console.error("Error checking user progress:", e);
                return { isRegistered: false, phase1: false, phase2: false };
            }
        }
        
        // --- NEW: Registration Screen ---

        function renderRegistrationScreen() {
            document.getElementById('content').innerHTML = `
                <div class="text-center p-8 bg-white rounded-xl card-shadow">
                    <h2 class="text-3xl font-extrabold text-dunkirk-blue mb-4">開始個人模擬：請填寫基本資訊</h2>
                    <p class="text-lg text-gray-700 mb-6">請輸入您的**班級**與**姓名**，以便儲存您的進度與結果。</p>
                    
                    <form id="registration-form" class="space-y-4 max-w-sm mx-auto">
                        <div>
                            <input type="text" id="reg-class" placeholder="您的班級 (例如: 高二丙班)" required 
                                class="w-full p-3 border border-gray-300 rounded-lg focus:ring-accent-orange focus:border-accent-orange">
                        </div>
                        <div>
                            <input type="text" id="reg-name" placeholder="您的姓名 (例如: 王小明)" required 
                                class="w-full p-3 border border-gray-300 rounded-lg focus:ring-accent-orange focus:border-accent-orange">
                        </div>
                        <button type="submit" class="
                            w-full bg-accent-orange text-white font-bold py-3 px-8 rounded-xl 
                            hover:bg-orange-700 transition-all duration-300 
                            shadow-lg hover:shadow-xl text-lg tracking-wider
                        ">
                            確認並進入歡迎畫面
                        </button>
                        <p id="reg-error" class="text-red-500 text-sm mt-3"></p>
                    </form>
                </div>
            `;

            document.getElementById('registration-form').addEventListener('submit', submitRegistration);
        }

        function submitRegistration(e) {
            e.preventDefault();
            const inputClass = document.getElementById('reg-class').value.trim();
            const inputName = document.getElementById('reg-name').value.trim();

            if (inputClass && inputName) {
                userName = inputName;
                userClass = inputClass;
                saveRegistrationData().then(() => {
                    updateUserInfoDisplay();
                    renderStartScreen();
                }).catch(e => {
                    document.getElementById('reg-error').textContent = '儲存資訊失敗，請重試。';
                    console.error("Registration save failed:", e);
                });
            } else {
                document.getElementById('reg-error').textContent = '班級與姓名皆為必填。';
            }
        }

        async function saveRegistrationData() {
            const data = {
                userName: userName,
                userClass: userClass,
                isRegistered: true
            };
            
            await setDoc(userChoicesRef, data, { merge: true });
        }


        // --- Start Screen ---

        function renderStartScreen() {
            currentPhase = 'start';
            document.getElementById('content').innerHTML = `
                <div class="text-center p-8 bg-white rounded-xl card-shadow">
                    <h2 class="text-3xl font-extrabold text-dunkirk-blue mb-4">歡迎來到敦克爾克沙灘</h2>
                    <p class="text-lg text-gray-700 mb-6">玩家：<span class="font-bold">${userClass} / ${userName}</span></p>
                    
                    <p class="text-lg text-gray-700 mb-6">這是一個道德與人性的抉擇模擬。你將扮演一位小型救援船船長，面臨時間和資源的極限挑戰。</p>
                    
                    <button id="start-dilemma-btn" class="
                        bg-accent-orange text-white font-bold py-4 px-8 rounded-xl 
                        hover:bg-orange-700 transition-all duration-300 
                        shadow-lg hover:shadow-xl transform hover:scale-105
                        text-xl tracking-wider
                    ">
                        點擊開始：進入船長困境
                    </button>

                    <p class="text-sm text-gray-500 mt-6">（注意：一旦開始，第一階段的限時挑戰將立即啟動。）</p>
                </div>
            `;
            document.getElementById('start-dilemma-btn').addEventListener('click', renderPhase1Dilemma);
        }

        // --- Module 1: Captain's Dilemma (船長困境) ---

        let savedCount = 0;
        let selectedPeople = {};
        let timerInterval;

        function renderPhase1Dilemma() {
            currentPhase = 'phase1';
            savedCount = 0;
            selectedPeople = {};
            
            const peopleHTML = PEOPLE_DATA.map(group =>
                Array(group.count).fill(0).map((_, i) =>
                    `<div id="${group.id}-${i+1}" data-type="${group.type}" data-label="${group.label}" class="person-card card-shadow bg-sand-color p-3 m-1 rounded-lg text-sm text-center select-none" draggable="true">
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

                <!-- Timer -->
                <div class="text-center mb-4">
                    <span id="timer" class="text-3xl font-extrabold text-red-600 p-2 rounded-lg bg-red-100 card-shadow">03:00</span>
                </div>

                <!-- Drop Zone (Boat) -->
                <div id="drop-zone" class="drop-zone rounded-xl p-3 mb-6 flex flex-wrap justify-center items-center bg-gray-100">
                    <p class="text-gray-500 text-lg">拖曳 10 位待救者到此區域 (船艙)</p>
                </div>

                <div class="flex justify-between items-center mb-4">
                    <h3 class="text-xl font-semibold">待救者 (沙灘上的 30 人)</h3>
                    <button id="submit-p1" class="px-4 py-2 bg-accent-orange text-white rounded-lg font-semibold hover:bg-orange-700 transition-colors hidden">提交選擇</button>
                </div>
                
                <!-- Person Cards (Drag Source) -->
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
                card.addEventListener('dragend', (e) => {
                    e.target.classList.remove('dragging');
                });
                card.addEventListener('click', (e) => {
                    if (e.currentTarget.parentElement.id === 'people-container' && savedCount < 10) {
                        moveCard(e.currentTarget, dropZone, peopleContainer);
                    } else if (e.currentTarget.parentElement.id === 'drop-zone') {
                        moveCard(e.currentTarget, peopleContainer, dropZone);
                    }
                });
            });

            dropZone.addEventListener('dragover', (e) => {
                e.preventDefault();
                dropZone.classList.add('drag-over');
            });
            dropZone.addEventListener('dragleave', () => {
                dropZone.classList.remove('drag-over');
            });
            dropZone.addEventListener('drop', (e) => {
                e.preventDefault();
                dropZone.classList.remove('drag-over');
                const id = e.dataTransfer.getData('text/plain');
                const card = document.getElementById(id);
                if (card && card.parentElement.id === 'people-container' && savedCount < 10) {
                    moveCard(card, dropZone, peopleContainer);
                }
            });

            submitButton.addEventListener('click', submitPhase1);

            function updateCapacityDisplay() {
                document.getElementById('capacity-display').textContent = 10 - savedCount;
                if (savedCount === 10) {
                    const placeholder = dropZone.querySelector('p');
                    if (placeholder) {
                        dropZone.removeChild(placeholder);
                    }
                    submitButton.classList.remove('hidden');
                } else {
                    if (!dropZone.querySelector('p') && dropZone.children.length === 0) {
                       dropZone.innerHTML = '<p class="text-gray-500 text-lg">拖曳 10 位待救者到此區域 (船艙)</p>';
                    }
                    submitButton.classList.add('hidden');
                }
            }

            function moveCard(card, destination, source) {
                if (destination.id === 'drop-zone') {
                    if (savedCount >= 10) return;
                    savedCount++;
                    selectedPeople[card.id] = { type: card.dataset.type, label: card.dataset.label };
                } else if (destination.id === 'people-container') {
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
                 if (!window.confirm("您還沒有拯救任何人。確定要以空船提交嗎？")) {
                     startTimer(1);
                     return;
                 }
            }

            const selection = Object.values(selectedPeople).reduce((acc, person) => {
                acc[person.type] = (acc[person.type] || 0) + 1;
                return acc;
            }, {});

            renderMoralPrincipleForm(selection);
        }

        function renderMoralPrincipleForm(selection) {
            const moralPrinciplesHTML = Object.entries(MORAL_PRINCIPLES).map(([key, principle]) => `
                <label class="block mb-2 p-3 bg-white rounded-lg card-shadow hover:bg-gray-50 transition-colors cursor-pointer">
                    <input type="checkbox" name="moral_principle" value="${key}" class="mr-3 accent-accent-orange">
                    ${principle}
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
                    <p class="mt-4">請選擇 **1 到 3 個**最能代表你決策依據的**道德原則**。</p>
                </div>
                
                <form id="moral-form" class="space-y-3">
                    ${moralPrinciplesHTML}
                    <button type="submit" class="w-full mt-6 px-4 py-3 bg-accent-orange text-white rounded-lg font-semibold text-lg hover:bg-orange-700 transition-colors">確認原則並進入下一階段</button>
                    <p id="error-message" class="text-red-500 text-center"></p>
                </form>
            `;

            document.getElementById('moral-form').addEventListener('submit', (e) => {
                e.preventDefault();
                const selected = Array.from(e.target.querySelectorAll('input[name="moral_principle"]:checked')).map(el => el.value);
                
                if (selected.length === 0 || selected.length > 3) {
                    document.getElementById('error-message').textContent = '請選擇 1 到 3 個道德原則！';
                    return;
                }

                savePhase1Data(selection, selected);
            });
        }

        async function savePhase1Data(selection, principles) {
            try {
                const savedPeopleCount = Object.entries(selection).reduce((acc, [type, count]) => {
                    if (count > 0) {
                        acc[type] = count;
                    }
                    return acc;
                }, {});

                await setDoc(userChoicesRef, {
                    userId: userId,
                    phase1Completed: true,
                    timestamp: new Date().toISOString(),
                    savedCount: savedCount,
                    savedPeople: savedPeopleCount,
                    moralPrinciples: principles
                }, { merge: true });

                currentPhase = 'phase2';
                renderPhase2();

            } catch (e) {
                console.error("Error saving phase 1 data:", e);
                document.getElementById('error-message').textContent = '儲存失敗，請檢查連線。';
            }
        }

        async function fetchWithExponentialBackoff(url, options, maxRetries = 5) {
            for (let i = 0; i < maxRetries; i++) {
                try {
                    const response = await fetch(url, options);
                    if (response.ok) {
                        return response;
                    }
                    if (response.status !== 429 && response.status < 500) {
                        throw new Error(`API returned status ${response.status}: ${await response.text()}`);
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
            const systemPrompt = `你就是蘇格拉底的靈魂。你的目標不是給予答案，而是根據學生提供的倫理和道德數據，提出一個深入、挑戰他們核心假設、偏見或矛盾的批判或反問。你的回應必須以繁體中文（Traditional Chinese）呈現，以一個有力的詰問開頭，並力求簡潔（不超過 150 字）。`;
            
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
                systemInstruction: {
                    parts: [{ text: systemPrompt }]
                },
            };
            
            try {
                const response = await fetchWithExponentialBackoff(
                    GEMINI_API_URL, 
                    {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    }
                );
                
                const result = await response.json();
                const text = result.candidates?.[0]?.content?.parts?.[0]?.text;
                return text || "蘇格拉底的靈魂正在沉思中，沒有給出明確的答案。";
                
            } catch (error) {
                console.error("Gemini API call failed:", error);
                return "（由於技術限制，蘇格拉底今日無法發言。請自行思考你提交的反思！）";
            }
        }


        function renderPhase2() {
            currentPhase = 'phase2';
            const hopeSceneHTML = SCENES.hope.map(scene => `
                <label class="flex items-center p-3 bg-white rounded-lg card-shadow hover:bg-gray-50 transition-colors cursor-pointer">
                    <input type="radio" name="hope_vote" value="${scene.id}" data-desc="${scene.description}" class="mr-3 accent-dunkirk-blue" required>
                    ${scene.description}
                </label>
            `).join('');

            const despairSceneHTML = SCENES.despair.map(scene => `
                <label class="flex items-center p-3 bg-white rounded-lg card-shadow hover:bg-gray-50 transition-colors cursor-pointer">
                    <input type="radio" name="despair_vote" value="${scene.id}" data-desc="${scene.description}" class="mr-3 accent-dunkirk-blue" required>
                    ${scene.description}
                </label>
            `).join('');

            document.getElementById('content').innerHTML = `
                <h2 class="text-2xl font-bold mb-4 text-dunkirk-blue">第二階段：無名英雄與電影反思（深度互動）</h2>
                <p class="mb-6 text-gray-600">請完成以下所有任務，將「敦克爾克精神」延伸到您的日常觀察，並進行深度道德反思。</p>
                
                <form id="phase2-form" class="space-y-6">
                    
                    <!-- Task 1: Unsung Hero (Individual Reflection) -->
                    <div class="p-4 bg-sand-color rounded-xl card-shadow">
                        <h3 class="text-xl font-semibold mb-3">任務一：現代的敦克爾克英雄 (個人反思)</h3>
                        <p class="mb-3 text-sm">請寫下一個您認為在日常生活中體現了「敦克爾克精神」(集體奉獻/堅守崗位) 的**無名英雄身份**及其**行為標籤**。</p>
                        <input type="text" id="hero-identity" placeholder="例如: 清潔隊員" required class="w-full p-2 border border-gray-300 rounded-lg mb-2">
                        <input type="text" id="hero-tag" placeholder="例如: #默默堅守" required class="w-full p-2 border border-gray-300 rounded-lg">
                    </div>

                    <!-- Task 2: Hope Vote + Reason -->
                    <div class="p-4 bg-white rounded-xl card-shadow">
                        <h3 class="text-xl font-semibold mb-3 text-green-700">任務二：希望時刻投票與理由</h3>
                        <p class="mb-4 text-sm">您認為電影中哪一個畫面最能代表**「希望」**？</p>
                        <div class="space-y-2">${hopeSceneHTML}</div>
                        <div class="mt-4">
                            <label for="hope-reason" class="block text-sm font-medium mb-1">請簡述您選擇此場景的理由 (50字內):</label>
                            <textarea id="hope-reason" rows="2" placeholder="說明您為何認為這是最能代表希望的時刻。" required class="w-full p-2 border border-gray-300 rounded-lg resize-none"></textarea>
                        </div>
                    </div>

                    <!-- Task 3: Despair Vote + Reason -->
                    <div class="p-4 bg-white rounded-xl card-shadow">
                        <h3 class="text-xl font-semibold mb-3 text-red-700">任務三：絕望時刻投票與理由</h3>
                        <p class="mb-4 text-sm">您認為電影中哪一個畫面最能代表**「絕望」**？</p>
                        <div class="space-y-2">${despairSceneHTML}</div>
                        <div class="mt-4">
                            <label for="despair-reason" class="block text-sm font-medium mb-1">請簡述您選擇此場景的理由 (50字內):</label>
                            <textarea id="despair-reason" rows="2" placeholder="說明您為何認為這是最能代表絕望的時刻。" required class="w-full p-2 border border-gray-300 rounded-lg resize-none"></textarea>
                        </div>
                    </div>
                    
                    <!-- Task 4: Ethical Reflection -->
                    <div class="p-4 bg-yellow-100/50 rounded-xl card-shadow">
                        <h3 class="text-xl font-semibold mb-3 text-dunkirk-blue">任務四：道德反思與矛盾 (魔鬼代言人挑戰)</h3>
                        <p class="mb-4 text-sm">請指出在**第一階段的抉擇**或**電影情境**中，你觀察到或思考到的任何**倫理矛盾，或潛在的偏見**。（例如：時間壓力下的不公、效用原則的殘酷性等）</p>
                        <textarea id="ethical-reflection" rows="4" placeholder="請寫下你的反思，這是課程討論的重點。" required class="w-full p-2 border border-gray-300 rounded-lg resize-none"></textarea>
                    </div>

                    <button type="submit" class="w-full px-4 py-3 bg-dunkirk-blue text-white rounded-lg font-semibold text-lg hover:bg-gray-700 transition-colors">提交反思結果並查看個人分析</button>
                    <p id="error-message-p2" class="text-red-500 text-center"></p>
                </form>
            `;

            document.getElementById('phase2-form').addEventListener('submit', (e) => {
                e.preventDefault();
                savePhase2Data();
            });
        }

        async function savePhase2Data() {
            try {
                const heroIdentity = document.getElementById('hero-identity').value.trim();
                const heroTag = document.getElementById('hero-tag').value.trim();
                const hopeChecked = document.querySelector('input[name="hope_vote"]:checked');
                const despairChecked = document.querySelector('input[name="despair_vote"]:checked');

                if (!heroIdentity || !heroTag || !hopeChecked || !despairChecked) {
                    document.getElementById('error-message-p2').textContent = '請完成所有欄位填寫！';
                    return;
                }

                const hopeSceneId = hopeChecked.value;
                const despairSceneId = despairChecked.value;
                const hopeSceneDescription = hopeChecked.dataset.desc;
                const despairSceneDescription = despairChecked.dataset.desc;

                const phase2Data = {
                    phase2Completed: true,
                    heroIdentity,
                    heroTag,
                    hopeSceneId: parseInt(hopeSceneId),
                    hopeSceneDescription: hopeSceneDescription,
                    hopeReason: document.getElementById('hope-reason').value.trim(),
                    despairSceneId: parseInt(despairSceneId),
                    despairSceneDescription: despairSceneDescription,
                    despairReason: document.getElementById('despair-reason').value.trim(),
                    ethicalReflection: document.getElementById('ethical-reflection').value.trim(),
                };
                
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
                
                socraticCritique = await generateSocraticCritique(phase2Data);

                await setDoc(userChoicesRef, {
                    ...phase2Data,
                    socraticCritique: socraticCritique,
                    timestampP2: new Date().toISOString(),
                }, { merge: true });

                currentPhase = 'results';
                renderResults();

            } catch (e) {
                console.error("Error saving phase 2 data or generating critique:", e);
                document.getElementById('error-message-p2').textContent = '提交失敗，請檢查網路連線或重試。';
                currentPhase = 'results';
                renderResults();
            }
        }

        // --- Module 3: Results & Analysis (Personal Only) ---

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
                const userData = docSnap.data();
                if (userData) {
                    socraticCritique = userData.socraticCritique || socraticCritique || 'AI 蘇格拉底的詰問未被載入。';

                    const p1Summary = Object.entries(userData.savedPeople || {}).map(([type, count]) => {
                        const label = PEOPLE_DATA.find(p => p.type === type)?.label || '未知';
                        return `<span class="inline-block bg-dunkirk-blue text-white text-sm px-3 py-1 m-1 rounded-full">${label} x ${count}</span>`;
                    }).join('') || '<p class="text-gray-500">無人被拯救。</p>';

                    const principleSummary = (userData.moralPrinciples || []).map(key => MORAL_PRINCIPLES[key]).join('；') || '未選擇原則';
                    
                    const heroSummary = `${userData.heroIdentity || '無'} (${userData.heroTag || '無'})`;
                    const hopeDesc = userData.hopeSceneDescription || '無';
                    const despairDesc = userData.despairSceneDescription || '無';
                    
                    document.getElementById('content').innerHTML = `
                        <h2 class="text-2xl font-bold mb-6 text-dunkirk-blue border-b-2 border-accent-orange pb-2">個人分析：你的抉擇總結</h2>
                        
                        <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
                            <!-- User's Reflection and Critique -->
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

                            <!-- LLM Critique (Socratic) -->
                            <div class="lg:col-span-1 p-6 bg-yellow-100 rounded-xl card-shadow border-t-8 border-yellow-500">
                                <h3 class="text-xl font-bold mb-3 text-yellow-800">蘇格拉底的詰問</h3>
                                <div class="text-base text-yellow-900 italic font-serif">
                                    <p>${socraticCritique}</p>
                                </div>
                            </div>
                        </div>
                    `;
                } else {
                    document.getElementById('content').innerHTML = `<p class="text-red-600 text-center">無法載入你的抉擇數據。</p>`;
                }
            }).catch(e => {
                console.error("Error fetching user choices for results:", e);
                document.getElementById('content').innerHTML = `<p class="text-red-600 text-center">載入錯誤：請檢查控制台。</p>`;
            });
        }

        window.onload = initFirebase;
    </script>
