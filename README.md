<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>一起來舔管管吧!</title>
    <!-- 載入 Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- 載入 Tone.js 用於預設音效 -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.min.js"></script>
    <!-- 新增：Cropper.js 裁切工具的 CSS -->
    <link href="https://cdnjs.cloudflare.com/ajax/libs/cropperjs/1.5.13/cropper.min.css" rel="stylesheet">
    <!-- 新增：Cropper.js 裁切工具的 JS -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/cropperjs/1.5.13/cropper.min.js"></script>
    <style>
        body, html {
            height: 100%;
            margin: 0;
            /* overflow: hidden; */ /* <-- 修正：移除此行，允許選單捲動 */
        }
        
        body {
            font-family: 'Inter', sans-serif;
            -webkit-tap-highlight-color: transparent;
            user-select: none;
            cursor: default;
        }

        /* 遊戲目標 (單一或多重) */
        .game-target {
            position: absolute;
            transition: top 0.1s ease-out, left 0.1s ease-out;
            border-radius: 50%; 
            object-fit: cover; 
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
            cursor: pointer;
        }
        .game-target:active {
            transform: scale(0.95);
        }

        /* 點擊特效圖片的樣式 */
        .click-effect-image {
            position: absolute;
            width: 50px; 
            height: 50px;
            object-fit: contain;
            pointer-events: none; 
            opacity: 0; 
            transition: opacity 0.3s ease-out, transform 0.3s ease-out; 
            transform: scale(0.5); 
            z-index: 1000; 
        }
        .click-effect-image.show {
            opacity: 1;
            transform: scale(1); 
        }
        
        /* 隱藏預設的檔案上傳按鈕 */
        input[type="file"] {
            display: none;
        }
        /* 自訂檔案上傳按鈕的樣式 */
        .custom-file-upload {
            border: 2px dashed #4A5568;
            display: inline-block;
            padding: 6px 12px;
            cursor: pointer;
            background-color: #2D3748;
            color: #E2E8F0;
            border-radius: 8px;
            transition: background-color 0.2s;
        }
        .custom-file-upload:hover {
            background-color: #4A5568;
        }

        /* 裁切視窗的樣式 */
        #cropper-modal {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.75);
            display: flex;
            align-items: center;
            justify-content: center;
            z-index: 2000; /* 確保在最上層 */
            opacity: 0;
            pointer-events: none;
            transition: opacity 0.3s ease;
        }
        #cropper-modal.show {
            opacity: 1;
            pointer-events: auto;
        }
        #cropper-container {
            max-width: 90vw;
            max-height: 80vh;
        }
        #cropper-image {
            display: block;
            max-width: 100%;
        }

        /* 遊戲主容器 */
        #game-container {
            display: flex;
            flex-direction: column;
            height: 100vh;
        }
        /* 頂部 HUD 面板 */
        #hud-container {
            flex-shrink: 0; /* 防止面板縮小 */
            background-color: #1F2937; /* 較深的灰色 */
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
            z-index: 100;
        }
        /* 遊戲區域 */
        #game-area {
            position: relative; /* 關鍵：讓目標相對於此區域定位 */
            flex-grow: 1; /* 填滿剩餘空間 */
            background-color: #111827; /* 最深的灰色 */
            overflow: hidden; /* 隱藏溢出的目標 */
        }
    </style>
</head>
<body class="bg-gray-900 text-white">

    <!-- 裁切視窗 (預設隱藏) -->
    <div id="cropper-modal" class="hidden">
        <div class="bg-gray-800 p-4 rounded-lg shadow-xl">
            <!-- 裁切圖片將顯示於此 -->
            <div id="cropper-container">
                <img id="cropper-image" src="">
            </div>
            <button id="crop-btn" class="w-full mt-4 bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg">
                裁切並使用
            </button>
            <button id="crop-cancel-btn" class="w-full mt-2 bg-gray-600 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded-lg">
                取消
            </button>
        </div>
    </div>

    <!-- 遊戲主選單 (全螢幕) -->
    <div id="menu-screen" class="flex items-center justify-center min-h-screen">
        <div class="text-center p-8 bg-gray-800 rounded-lg shadow-2xl max-w-md w-full">
            <h1 class="text-4xl font-bold mb-6">反應速度訓練</h1>

            <!-- 圖片上傳區 -->
            <div class="grid grid-cols-2 gap-4 mb-6 text-sm">
                <div>
                    <label for="target-upload" class="custom-file-upload">
                        上傳「點擊目標」
                    </label>
                    <input id="target-upload" type="file" accept="image/*">
                    <img id="target-preview" src="https://i.imgur.com/gK2oH1N.png" class="w-16 h-16 rounded-full mx-auto mt-2 object-cover">
                </div>
                <div>
                    <label for="effect-upload" class="custom-file-upload">
                        上傳「點擊特效」
                    </label>
                    <input id="effect-upload" type="file" accept="image/*">
                    <img id="effect-preview" src="https://i.imgur.com/8Q0G3Jk.png" class="w-16 h-16 object-contain mx-auto mt-2">
                </div>
            </div>

            <!-- 上傳音效按鈕 -->
            <div class="mb-4 text-sm">
                <label for="sound-upload" class="custom-file-upload w-full">
                    上傳「點擊音效」(.mp3, .wav)
                </label>
                <input id="sound-upload" type="file" accept="audio/*">
                <p id="sound-preview" class="text-gray-400 mt-2 h-6">使用預設「啵」聲</p>
            </div>

            <!-- Google Drive 連結 -->
            <div class="mb-6 text-sm text-gray-400 border border-gray-600 p-3 rounded-lg">
                <p class="font-bold text-white mb-2">尋找更多圖片？</p>
                <a href="https://drive.google.com/drive/folders/1J014ESx-HFTpfvkw_Ym-6slYfEvMkwm8?usp=drive_link" 
                   target="_blank" 
                   class="text-blue-400 hover:underline">
                   點此瀏覽建議的圖片庫
                </a>
                <p class="mt-2">（提醒：您需要從該連結下載圖片，然後再使用上方的「上傳」按鈕）</p>
            </div>

            <!-- 新增：遊戲模式選擇 -->
            <div class="mb-4">
                <label class="block mb-2 text-lg">遊戲模式：</label>
                <div class="flex justify-center gap-4">
                    <button id="mode-single-btn" class="flex-1 bg-blue-600 text-white font-bold py-2 px-4 rounded-lg">
                        單一目標
                    </button>
                    <button id="mode-multi-btn" class="flex-1 bg-gray-600 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded-lg">
                        多重目標
                    </button>
                </div>
            </div>

            <!-- 計時模式 -->
            <div class="mb-4">
                <label for="time-input" class="block mb-2 text-lg">設定時間（秒）：</label>
                <input id="time-input" type="number" value="30" min="5" class="w-full p-2 text-center text-black rounded-md text-2xl">
            </div>
            <button id="start-timed-btn" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-lg text-xl mb-4 transition-colors">
                開始計時模式
            </button>
            
            <!-- 無限模式 -->
            <button id="start-infinite-btn" class="w-full bg-green-600 hover:bg-green-700 text-white font-bold py-3 px-6 rounded-lg text-xl transition-colors">
                開始無限模式
            </button>
        </div>
    </div>

    <!-- 遊戲進行中 (包含 HUD 和 遊戲區域) -->
    <div id="game-container" class="hidden">
        
        <!-- 頂部 HUD 面板 -->
        <div id="hud-container" class="w-full p-4 flex justify-between items-center">
            <!-- 分數 -->
            <div class="flex items-center space-x-2">
                <img src="" alt="分數圖示" class="w-10 h-10 object-contain score-icon">
                <p id="score-display" class="text-4xl font-extrabold text-yellow-400">0</p>
            </div>
            
            <!-- 計時器 -->
            <div id="timer-container" class="hidden text-center">
                <span class="text-gray-400 text-xl">時間</span>
                <p id="timer-display" class="text-4xl font-extrabold text-red-500">30</p>
            </div>
            
            <!-- 離開按鈕 -->
            <button id="exit-btn" class="bg-gray-600 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded-lg transition-colors">
                返回選單
            </button>
        </div>

        <!-- 遊戲區域 -->
        <div id="game-area">
            <!-- 單一目標 (預設) -->
            <img id="target-image" 
                 src="" 
                 alt="點擊目標" 
                 class="game-target w-24 h-24 md:w-32 md:h-32"
                 draggable="false"
                 style="display: none;"> <!-- 預設隱藏 -->
            
            <!-- 多重目標會被動態加入到這裡 -->
        </div>
    </div>


    <!-- 遊戲結束畫面 (全螢幕) -->
    <div id="game-over-screen" class="hidden items-center justify-center min-h-screen">
        <div class="text-center p-8 bg-gray-800 rounded-lg shadow-2xl">
            <h2 class="text-3xl font-bold mb-2">時間到！</h2>
            <p class="text-gray-400 text-xl mb-4">您的最終分數：</p>
            <!-- 最終分數 -->
            <div class="flex items-center justify-center space-x-2 mb-8">
                <img src="" alt="分數圖示" class="w-16 h-16 object-contain score-icon">
                <p id="final-score" class="text-7xl font-extrabold text-yellow-400">0</p>
            </div>
            <button id="back-to-menu-btn" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-lg text-xl transition-colors">
                返回主選單
            </button>
        </div>
    </div>


    <script>
        // -----------------------------------------------------
        // 預設圖片 URL
        // -----------------------------------------------------
        const DEFAULT_TARGET_URL = 'https://i.imgur.com/gK2oH1N.png';
        const DEFAULT_EFFECT_URL = 'https://i.imgur.com/8Q0G3Jk.png';

        // -----------------------------------------------------
        // 當前使用的圖片 (Data URL 或 預設 URL)
        // -----------------------------------------------------
        let currentTargetImageUrl = DEFAULT_TARGET_URL;
        let currentClickEffectImageUrl = DEFAULT_EFFECT_URL;

        // -----------------------------------------------------
        // DOM 元素
        // -----------------------------------------------------
        const menuScreen = document.getElementById('menu-screen');
        const gameContainer = document.getElementById('game-container'); // 遊戲主容器
        const hudContainer = document.getElementById('hud-container'); // 頂部 HUD
        const gameArea = document.getElementById('game-area');       // 遊戲區域
        const gameOverScreen = document.getElementById('game-over-screen');
        
        const targetImage = document.getElementById('target-image'); // 單一目標
        const timeInput = document.getElementById('time-input');
        
        const startTimedBtn = document.getElementById('start-timed-btn');
        const startInfiniteBtn = document.getElementById('start-infinite-btn');
        const exitBtn = document.getElementById('exit-btn');
        const backToMenuBtn = document.getElementById('back-to-menu-btn');
        
        const scoreDisplay = document.getElementById('score-display');
        const timerContainer = document.getElementById('timer-container');
        const timerDisplay = document.getElementById('timer-display');
        const finalScore = document.getElementById('final-score');
        
        const scoreIcons = document.querySelectorAll('.score-icon');

        // 圖片上傳元素
        const targetUpload = document.getElementById('target-upload');
        const effectUpload = document.getElementById('effect-upload');
        const targetPreview = document.getElementById('target-preview');
        const effectPreview = document.getElementById('effect-preview');

        // 音效上傳元素
        const soundUpload = document.getElementById('sound-upload');
        const soundPreview = document.getElementById('sound-preview');

        // 裁切視窗元素
        const cropperModal = document.getElementById('cropper-modal');
        const cropperImage = document.getElementById('cropper-image');
        const cropBtn = document.getElementById('crop-btn');
        const cropCancelBtn = document.getElementById('crop-cancel-btn');

        // 遊戲模式按鈕
        const modeSingleBtn = document.getElementById('mode-single-btn');
        const modeMultiBtn = document.getElementById('mode-multi-btn');

        // -----------------------------------------------------
        // 遊戲狀態變數
        // -----------------------------------------------------
        let score = 0;
        let timer = 0;
        let timerInterval = null;
        let timeMode = 'infinite'; 
        let gameMode = 'single'; // 'single' 或 'multi'
        const multiTargetCount = 5; // 多目標模式的數量

        // -----------------------------------------------------
        // 音效設定
        // -----------------------------------------------------
        let popSound; // Tone.js 預設音效
        if (typeof Tone !== 'undefined') {
            popSound = new Tone.MembraneSynth().toDestination();
        } else {
            console.error("Tone.js 載入失敗。");
        }

        let currentClickSoundUrl = null; // 用於儲存自訂音效的 Data URL

        async function initAudio() {
            if (popSound && Tone.context.state !== 'running') {
                try {
                    await Tone.start();
                    console.log("音訊已準備就緒！");
                } catch (e) {
                    console.error("啟動音訊時發生錯誤:", e);
                }
            }
        }
        
        // -----------------------------------------------------
        // 檔案上傳與裁切
        // -----------------------------------------------------
        let cropper;
        let currentUploadType = null; // 'target' 或 'effect'
        let currentUploadFileType = 'image/png'; // 預設
        
        function handleImageUpload(e, uploadType) {
            const file = e.target.files[0];
            if (!file) return;

            currentUploadType = uploadType;
            currentUploadFileType = file.type;

            const reader = new FileReader();
            reader.onload = (event) => {
                cropperImage.src = event.target.result;
                cropperModal.classList.remove('hidden');
                cropperModal.classList.add('show');

                if (cropper) {
                    cropper.destroy();
                }

                let options = {
                    viewMode: 1,
                    background: false,
                    autoCropArea: 0.8
                };
                
                if (uploadType === 'target') {
                    options.aspectRatio = 1 / 1; // 目標強制 1:1
                }

                cropper = new Cropper(cropperImage, options);
            };
            reader.readAsDataURL(file);
            e.target.value = null;
        }

        targetUpload.addEventListener('change', (e) => handleImageUpload(e, 'target'));
        effectUpload.addEventListener('change', (e) => handleImageUpload(e, 'effect'));

        cropBtn.addEventListener('click', () => {
            if (!cropper) return;
            const croppedDataUrl = cropper.getCroppedCanvas({
                maxWidth: 400, 
                maxHeight: 400,
            }).toDataURL(currentUploadFileType);

            if (currentUploadType === 'target') {
                currentTargetImageUrl = croppedDataUrl;
                targetPreview.src = croppedDataUrl;
            } else if (currentUploadType === 'effect') {
                currentClickEffectImageUrl = croppedDataUrl;
                effectPreview.src = croppedDataUrl;
            }

            cropperModal.classList.add('hidden');
            cropperModal.classList.remove('show');
            cropper.destroy();
            cropper = null;
        });

        cropCancelBtn.addEventListener('click', () => {
            cropperModal.classList.add('hidden');
            cropperModal.classList.remove('show');
            if (cropper) {
                cropper.destroy();
            }
            cropper = null;
        });

        soundUpload.addEventListener('change', (e) => {
            const file = e.target.files[0];
            if (file && file.type.startsWith('audio/')) {
                const reader = new FileReader();
                reader.onload = (event) => {
                    currentClickSoundUrl = event.target.result;
                    soundPreview.textContent = file.name; 
                };
                reader.readAsDataURL(file);
            }
        });

        // -----------------------------------------------------
        // 遊戲核心功能
        // -----------------------------------------------------

        function showScreen(screenName) {
            // 先隱藏所有全螢幕
            menuScreen.style.display = 'none';
            gameContainer.style.display = 'none';
            gameOverScreen.style.display = 'none';

            if (screenName === 'menu') {
                menuScreen.style.display = 'flex';
                // 重設預覽
                targetPreview.src = currentTargetImageUrl;
                effectPreview.src = currentClickEffectImageUrl;
                if (currentClickSoundUrl) {
                    soundPreview.textContent = soundUpload.files[0]?.name || "已載入自訂音效";
                } else {
                    soundPreview.textContent = "使用預設「啵」聲";
                }
            } else if (screenName === 'game') {
                gameContainer.style.display = 'flex'; // 使用 flex
            } else if (screenName === 'game_over') {
                gameOverScreen.style.display = 'flex';
            }
        }

        // 移動目標 (現在接收一個 DOM 元素作為參數)
        function moveTarget(elementToMove) {
            // 使用 gameArea 的尺寸
            const gameAreaWidth = gameArea.clientWidth;
            const gameAreaHeight = gameArea.clientHeight;
            const targetWidth = elementToMove.offsetWidth;
            const targetHeight = elementToMove.offsetHeight;

            const maxX = gameAreaWidth - targetWidth;
            const maxY = gameAreaHeight - targetHeight;

            const randomX = Math.floor(Math.random() * maxX);
            const randomY = Math.floor(Math.random() * maxY); 

            elementToMove.style.left = `${randomX}px`;
            elementToMove.style.top = `${randomY}px`;
        }

        function showClickEffect(x, y) {
            const effect = document.createElement('img');
            effect.src = currentClickEffectImageUrl; 
            effect.classList.add('click-effect-image');
            // 修正座標，相對於 body
            effect.style.left = `${x - 25}px`; 
            effect.style.top = `${y - 25}px`;  
            document.body.appendChild(effect); // 附加到 body 上

            requestAnimationFrame(() => {
                effect.classList.add('show');
            });
            
            setTimeout(() => {
                effect.classList.remove('show');
                effect.addEventListener('transitionend', () => effect.remove());
            }, 300); 
        }

        function playClickSound() {
            if (currentClickSoundUrl) {
                const audio = new Audio(currentClickSoundUrl);
                audio.play().catch(e => console.error("播放自訂音效失敗:", e));
            } else if (popSound) {
                try {
                    popSound.triggerAttackRelease("C3", "8n");
                } catch (e) { console.error("播放預設音效錯誤:", e); }
            }
        }

        // 點擊事件 (現在接收被點擊的目標)
        function onTargetClick(event, targetElement) {
            playClickSound();

            score++;
            scoreDisplay.textContent = score;
            
            const clickX = event.clientX || (event.touches && event.touches[0].clientX);
            const clickY = event.clientY || (event.touches && event.touches[0].clientY);
            
            if (clickX !== undefined && clickY !== undefined) {
                 showClickEffect(clickX, clickY);
            }

            // 只移動被點擊的那個目標
            moveTarget(targetElement);
        }

        function updateTimer() {
            timer--;
            timerDisplay.textContent = timer;
            if (timer <= 0) {
                endGame();
            }
        }

        // 清除所有目標 (多重或單一)
        function clearTargets() {
            // 移除所有多重目標
            const multiTargets = gameArea.querySelectorAll('.multi-target');
            multiTargets.forEach(target => target.remove());
            
            // 隱藏單一目標
            targetImage.style.display = 'none';
        }

        // 產生多重目標
        function spawnMultiTargets(count) {
            for (let i = 0; i < count; i++) {
                const newTarget = document.createElement('img');
                newTarget.src = currentTargetImageUrl;
                newTarget.alt = "點擊目標";
                newTarget.draggable = false;
                // 添加 'multi-target' 標記
                newTarget.className = "game-target multi-target w-24 h-24";
                
                // 綁定事件
                newTarget.addEventListener('mousedown', (e) => onTargetClick(e, newTarget)); 
                newTarget.addEventListener('touchstart', (e) => {
                    e.preventDefault(); 
                    onTargetClick(e.touches[0] ? e.touches[0] : e, newTarget);
                });
                
                gameArea.appendChild(newTarget);
                moveTarget(newTarget); // 移動到初始位置
            }
        }

        async function startGame(mode) {
            await initAudio(); 
            
            timeMode = mode;
            score = 0;
            scoreDisplay.textContent = 0;
            
            // 套用使用者上傳的圖片
            targetImage.src = currentTargetImageUrl;
            scoreIcons.forEach(icon => {
                icon.src = currentClickEffectImageUrl;
            });

            showScreen('game');
            clearTargets(); // 清除上一局的目標

            // 根據遊戲模式建立目標
            if (gameMode === 'single') {
                targetImage.style.display = 'block';
                moveTarget(targetImage); // 移動單一目標
            } else { // 'multi'
                spawnMultiTargets(multiTargetCount);
            }

            // 設定計時器
            if (timeMode === 'timed') {
                timer = parseInt(timeInput.value, 10) || 30;
                timerDisplay.textContent = timer;
                timerContainer.classList.remove('hidden');
                
                if (timerInterval) clearInterval(timerInterval);
                timerInterval = setInterval(updateTimer, 1000);

            } else { // 'infinite'
                timerContainer.classList.add('hidden');
                if (timerInterval) clearInterval(timerInterval);
                timerInterval = null;
            }
        }

        function endGame() {
            clearInterval(timerInterval);
            timerInterval = null;
            finalScore.textContent = score;
            showScreen('game_over');
        }

        function returnToMenu() {
            if (timerInterval) clearInterval(timerInterval);
            timerInterval = null;
            clearTargets(); // 離開時清除目標
            showScreen('menu');
        }

        // -----------------------------------------------------
        // 事件綁定
        // -----------------------------------------------------
        startTimedBtn.addEventListener('click', () => startGame('timed'));
        startInfiniteBtn.addEventListener('click', () => startGame('infinite'));
        
        // 綁定單一目標的點擊
        targetImage.addEventListener('mousedown', (e) => onTargetClick(e, targetImage)); 
        targetImage.addEventListener('touchstart', (e) => {
            e.preventDefault(); 
            onTargetClick(e.touches[0] ? e.touches[0] : e, targetImage);
        });

        exitBtn.addEventListener('click', returnToMenu);
        backToMenuBtn.addEventListener('click', returnToMenu);

        // 模式選擇按鈕
        modeSingleBtn.addEventListener('click', () => {
            gameMode = 'single';
            modeSingleBtn.classList.remove('bg-gray-600', 'hover:bg-gray-700');
            modeSingleBtn.classList.add('bg-blue-600');
            
            modeMultiBtn.classList.add('bg-gray-600', 'hover:bg-gray-700');
            modeMultiBtn.classList.remove('bg-blue-600');
        });
        modeMultiBtn.addEventListener('click', () => {
            gameMode = 'multi';
            modeMultiBtn.classList.remove('bg-gray-600', 'hover:bg-gray-700');
            modeMultiBtn.classList.add('bg-blue-600');
            
            modeSingleBtn.classList.add('bg-gray-600', 'hover:bg-gray-700');
            modeSingleBtn.classList.remove('bg-blue-600');
        });


        // 初始顯示主選單並設定預覽
        showScreen('menu');
    </script>
</body>
</html>
