# EatTheLand.github.io
EatTheLand.github.io
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>땅따먹기 게임</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
        }
        canvas {
            display: block;
        }
        #stageSelect {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: #fdf5ea; /* 스테이지 선택 화면 배경색 */
            color: #7b4626;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 10;
        }
        .stage-button {
            background: #7b4626; /* 버튼 배경색 */
            color: #fdf5ea; /* 버튼 글씨 색상 */
            border: none;
            padding: 10px 20px;
            margin: 10px;
            font-size: 20px;
            cursor: pointer;
            border-radius: 20px; /* 좌우 라운드 처리 */
        }
        .stage-button:disabled {
            background: #888;
            cursor: not-allowed;
        }
        #retryButton {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: #444;
            color: white;
            border: none;
            padding: 15px 30px;
            font-size: 20px;
            cursor: pointer;
            display: none;
            z-index: 20;
        }
        #gameInfo {
            position: absolute;
            top: 20px;
            left: 20px;
            color: white;
            font-size: 20px;
            z-index: 20;
        }
    </style>
</head>
<body>
    <div id="stageSelect">
        <h1>스테이지 선택</h1>
        <button class="stage-button" data-stage="1">스테이지 1</button>
        <button class="stage-button" data-stage="2" disabled>스테이지 2</button>
        <button class="stage-button" data-stage="3" disabled>스테이지 3</button>
        <button class="stage-button" data-stage="4" disabled>스테이지 4</button>
        <button class="stage-button" data-stage="5" disabled>스테이지 5</button>
    </div>
    <button id="retryButton">다시하기</button>
    <div id="gameInfo" style="display: none;">
        <p>남은 타일: <span id="remainingTiles">0</span></p>
        <p>점령률: <span id="capturePercentage">0%</span></p>
    </div>
    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        const stages = [
            { id: 1, enemySpeed: 2, size: 200, playerSize: 10, enemySize: 15, numEnemies: 1 },
            { id: 2, enemySpeed: 3, size: 300, playerSize: 10, enemySize: 20, numEnemies: 1 },
            { id: 3, enemySpeed: 4, size: 500, playerSize: 10, enemySize: 25, numEnemies: 2 },
            { id: 4, enemySpeed: 5, size: 700, playerSize: 10, enemySize: 30, numEnemies: 3 },
            { id: 5, enemySpeed: 6, size: 1000, playerSize: 10, enemySize: 35, numEnemies: 4 },
        ];

        let currentStageIndex = 0;
        let captured = [];
        let keys = {};
        let gameRunning = false;
        let enemies = [];
        let player = { x: canvas.width / 2, y: canvas.height / 2, size: 10, speed: 5 };
        
        // Updated colors
        const COLORS = { 
            player: '#fde4ac', 
            enemy: '#fc5e45', 
            trail: 'green', 
            captured: '#9aaa6c', 
            stage: 'white', 
            outside: '#fdf5ea', /* 외곽 배경색 */
            border: '#7b4626' /* 외곽선 색상 */
        };

        window.addEventListener('keydown', (e) => (keys[e.key] = true));
        window.addEventListener('keyup', (e) => (keys[e.key] = false));

        function isCollision(x1, y1, x2, y2, size1, size2) {
            return Math.hypot(x1 - x2, y1 - y2) < (size1 + size2) / 2;
        }

        function moveEnemy(stageX, stageY, stageSize, enemy) {
            enemy.x += enemy.speedX;
            enemy.y += enemy.speedY;

            // Wall collision detection
            if (enemy.x <= stageX || enemy.x >= stageX + stageSize - enemy.size) {
                enemy.speedX *= -1;
            }
            if (enemy.y <= stageY || enemy.y >= stageY + stageSize - enemy.size) {
                enemy.speedY *= -1;
            }
        }

        function movePlayer() {
            if (keys['ArrowUp'] || keys['w']) player.y -= player.speed;
            if (keys['ArrowDown'] || keys['s']) player.y += player.speed;
            if (keys['ArrowLeft'] || keys['a']) player.x -= player.speed;
            if (keys['ArrowRight'] || keys['d']) player.x += player.speed;

            const stageSize = stages[currentStageIndex].size;
            const stageX = (canvas.width - stageSize) / 2;
            const stageY = (canvas.height - stageSize) / 2;

            player.x = Math.max(stageX, Math.min(stageX + stageSize - player.size, player.x));
            player.y = Math.max(stageY, Math.min(stageY + stageSize - player.size, player.y));
        }

        function loadStage(stageIndex) {
            currentStageIndex = stageIndex;
            const stage = stages[stageIndex];
            enemies = [];
            for (let i = 0; i < stage.numEnemies; i++) {
                enemies.push({
                    x: Math.random() * stage.size + (canvas.width - stage.size) / 2,
                    y: Math.random() * stage.size + (canvas.height - stage.size) / 2,
                    size: stage.enemySize,
                    speedX: stage.enemySpeed,
                    speedY: stage.enemySpeed
                });
            }
            player.size = 10; // All stages set player size to 10
            captured = new Array(stage.size * stage.size).fill(false); // Initialize all tiles as unoccupied
            player.x = canvas.width / 2;
            player.y = canvas.height / 2;

            const stageSize = stages[stageIndex].size;
            const stageX = (canvas.width - stageSize) / 2;
            const stageY = (canvas.height - stageSize) / 2;

            gameRunning = true;
            document.getElementById('retryButton').style.display = 'none'; // Hide retry button when starting a new stage
            document.getElementById('gameInfo').style.display = 'block'; // Show game info
            gameLoop();
        }

        function gameLoop() {
            if (!gameRunning) return;

            const stageSize = stages[currentStageIndex].size;
            const stageX = (canvas.width - stageSize) / 2;
            const stageY = (canvas.height - stageSize) / 2;

            // Draw background
            ctx.fillStyle = COLORS.outside;
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            ctx.fillStyle = COLORS.stage;
            ctx.fillRect(stageX, stageY, stageSize, stageSize);

            // Draw border around the stage
            ctx.strokeStyle = COLORS.border;
            ctx.lineWidth = 2;
            ctx.strokeRect(stageX, stageY, stageSize, stageSize);

            // Move player
            movePlayer();

            // Calculate player's grid position
            const playerGridX = Math.floor((player.x - stageX) / player.size);
            const playerGridY = Math.floor((player.y - stageY) / player.size);
            const capturedIndex = playerGridY * stageSize + playerGridX;

            // Capture the tile if it's not already captured
            if (!captured[capturedIndex]) {
                captured[capturedIndex] = true;
            }

            // Move each enemy
            enemies.forEach(enemy => {
                moveEnemy(stageX, stageY, stageSize, enemy);
            });

            // Collision check with enemies
            for (let enemy of enemies) {
                if (isCollision(player.x, player.y, enemy.x, enemy.y, player.size, enemy.size)) {
                    document.getElementById('retryButton').style.display = 'block';
                    gameRunning = false;
                    return;
                }
            }

            // Calculate captured ratio based on remaining tiles inside the stage
            let totalStageTiles = 0;
            let remainingTiles = 0;

            captured.forEach((capturedCell, index) => {
                const x = (index % stageSize) * player.size + stageX;
                const y = Math.floor(index / stageSize) * player.size + stageY;

                // Only count tiles within the stage bounds (white area)
                if (x >= stageX && x < stageX + stageSize && y >= stageY && y < stageY + stageSize) {
                    totalStageTiles++;
                    if (!capturedCell) remainingTiles++;
                }
            });

            const capturedTiles = totalStageTiles - remainingTiles;

            // Achievement percentage formula
            const achievementPercentage = ((100 / totalStageTiles) * capturedTiles).toFixed(2);

            // Update game info
            document.getElementById('remainingTiles').textContent = remainingTiles;
            document.getElementById('capturePercentage').textContent = achievementPercentage + '%';

            // Check for stage clear
            if (remainingTiles <= totalStageTiles * 0.3) { // 70% of the unoccupied tiles
                alert(`스테이지 ${stages[currentStageIndex].id} 클리어!`);
                gameRunning = false;

                // Unlock next stage
                if (currentStageIndex + 1 < stages.length) {
                    document.querySelector(`.stage-button[data-stage="${currentStageIndex + 2}"]`).disabled = false;
                }
                document.getElementById('stageSelect').style.display = 'flex';
                return;
            }

            // Draw captured tiles
            ctx.fillStyle = COLORS.captured;
            captured.forEach((capturedCell, index) => {
                if (capturedCell) {
                    const x = (index % stageSize) * player.size + stageX;
                    const y = Math.floor(index / stageSize) * player.size + stageY;
                    ctx.fillRect(x, y, player.size, player.size);
                }
            });

            // Draw player
            ctx.fillStyle = COLORS.player;
            ctx.fillRect(player.x, player.y, player.size, player.size);

            // Draw enemies
            enemies.forEach(enemy => {
                ctx.fillStyle = COLORS.enemy;
                ctx.fillRect(enemy.x, enemy.y, enemy.size, enemy.size);
            });

            requestAnimationFrame(gameLoop);
        }

        // Stage selection button
        document.querySelectorAll('.stage-button').forEach((button) => {
            button.addEventListener('click', (e) => {
                const stageIndex = parseInt(e.target.dataset.stage, 10) - 1;
                document.getElementById('stageSelect').style.display = 'none';
                loadStage(stageIndex);
            });
        });

        // Retry button
        document.getElementById('retryButton').addEventListener('click', () => {
            loadStage(currentStageIndex); // Retry the current stage
        });
    </script>
</body>
</html>
