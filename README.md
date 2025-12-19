<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>병아리의 모험 - 10단계 미션</title>
    <style>
        body { margin: 0; background: #222; display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100vh; color: white; font-family: 'Courier New', Courier, monospace; overflow: hidden; }
        canvas { background: #87CEEB; border: 4px solid #fff; box-shadow: 0 0 20px rgba(0,0,0,0.5); image-rendering: pixelated; }
        .ui { margin-bottom: 10px; text-align: center; }
        .stats { display: flex; gap: 20px; font-size: 20px; font-weight: bold; }
        .controls { margin-top: 10px; font-size: 14px; color: #aaa; }
    </style>
</head>
<body>

    <div class="ui">
        <h1>🐥 병아리의 모험 🐥</h1>
        <div class="stats">
            <div>LEVEL: <span id="levelDisplay">1</span> / 10</div>
            <div>COINS: <span id="coinDisplay">0</span> / 5</div>
        </div>
    </div>

    <canvas id="gameCanvas" width="800" height="400"></canvas>

    <div class="controls">
        이동: 방향키(←, →) | 점프: Z 또는 Space | 벽타기: 벽에서 방향키 유지 (최대 2초)
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const levelDisplay = document.getElementById('levelDisplay');
        const coinDisplay = document.getElementById('coinDisplay');

        // 게임 설정
        const GRAVITY = 0.5;
        const JUMP_FORCE = -10;
        const MOVE_SPEED = 4;
        const CLIMB_TIME_LIMIT = 120; // 60fps 기준 2초

        let currentLevel = 1;
        let coinsCollected = 0;
        let gameState = "PLAY"; // PLAY, SUCCESS, GAMEOVER

        // 플레이어 객체
        const player = {
            x: 50, y: 300, w: 30, h: 30,
            vx: 0, vy: 0,
            isGrounded: false,
            isClimbing: false,
            climbTimer: 0,
            facing: 1, // 1: 우, -1: 좌
            reset(startX, startY) {
                this.x = startX; this.y = startY;
                this.vx = 0; this.vy = 0;
                this.climbTimer = 0;
                this.isClimbing = false;
            }
        };

        // 키 입력 관리
        const keys = {};
        window.onkeydown = (e) => keys[e.code] = true;
        window.onkeyup = (e) => keys[e.code] = false;

        // 맵 데이터 생성 (총 10개)
        function getLevelData(lvl) {
            const platforms = [
                {x: 0, y: 380, w: 2000, h: 20}, // 바닥
            ];
            const obstacles = []; // 가시
            const enemies = [];   // 농부
            const coins = [];

            // 난이도별 자동 생성 로직
            for(let i=1; i<=5; i++) {
                coins.push({x: 200 * i + (lvl * 20), y: 300 - (Math.sin(i)*50), w: 20, h: 20, collected: false});
            }

            // 레벨별 발판 및 장애물 배치 (점진적 난이도)
            for(let i=1; i<lvl + 2; i++) {
                platforms.push({x: 300 * i, y: 380 - (i * 40), w: 150, h: 20});
                if(lvl > 2) obstacles.push({x: 350 * i + 50, y: 360 - (i * 40), w: 30, h: 20}); // 가시
                if(lvl > 4) enemies.push({x: 400 * i, y: 340 - (i * 40), w: 30, h: 40, range: 100, startX: 400 * i, dir: 1});
            }

            return { platforms, obstacles, enemies, coins };
        }

        let map = getLevelData(currentLevel);

        function update() {
            if (gameState !== "PLAY") return;

            // 1. 좌우 이동
            if (keys['ArrowRight']) { player.vx = MOVE_SPEED; player.facing = 1; }
            else if (keys['ArrowLeft']) { player.vx = -MOVE_SPEED; player.facing = -1; }
            else { player.vx *= 0.8; }

            // 2. 중력 및 수직 이동
            if (!player.isClimbing) {
                player.vy += GRAVITY;
            }
            player.x += player.vx;
            player.y += player.vy;

            // 3. 바닥/발판 충돌 감지
            player.isGrounded = false;
            let onWall = false;

            map.platforms.forEach(p => {
                // 발판 위 충돌
                if (player.x < p.x + p.w && player.x + player.w > p.x &&
                    player.y + player.h > p.y && player.y + player.h < p.y + p.h + 10 && player.vy >= 0) {
                    player.y = p.y - player.h;
                    player.vy = 0;
                    player.isGrounded = true;
                    player.climbTimer = 0; // 바닥에 닿으면 벽타기 초기화
                }

                // 벽 충돌 (벽타기 로직)
                if (player.x + player.w >= p.x && player.x <= p.x + p.w &&
                    player.y + player.h > p.y && player.y < p.y + p.h) {
                    if (!player.isGrounded && (keys['ArrowRight'] || keys['ArrowLeft'])) {
                        onWall = true;
                    }
                }
            });

            // 4. 벽타기 처리 (2초 제한)
            if (onWall && player.climbTimer < CLIMB_TIME_LIMIT) {
                player.isClimbing = true;
                player.climbTimer++;
                player.vy = keys['ArrowUp'] ? -2 : 0.5; // 매달리기 또는 천천히 하강
            } else {
                player.isClimbing = false;
            }

            // 5. 점프
            if ((keys['Space'] || keys['KeyZ']) && (player.isGrounded || (onWall && player.climbTimer < CLIMB_TIME_LIMIT))) {
                player.vy = JUMP_FORCE;
                player.isGrounded = false;
                if(onWall) player.climbTimer += 20; // 벽점프 시 패널티
            }

            // 6. 장애물 충돌 (가시, 농부)
            map.obstacles.forEach(o => {
                if (checkRectCollision(player, o)) die();
            });

            map.enemies.forEach(e => {
                // 농부 움직임
                e.x += e.dir * 2;
                if (Math.abs(e.x - e.startX) > e.range) e.dir *= -1;
                if (checkRectCollision(player, e)) die();
            });

            // 7. 코인 수집
            map.coins.forEach(c => {
                if (!c.collected && checkRectCollision(player, c)) {
                    c.collected = true;
                    coinsCollected++;
                    coinDisplay.innerText = coinsCollected;
                }
            });

            // 8. 클리어 조건
            if (coinsCollected >= 5) {
                nextLevel();
            }

            // 화면 밖으로 떨어지면 사망
            if (player.y > canvas.height) die();
        }

        function checkRectCollision(r1, r2) {
            return r1.x < r2.x + r2.w && r1.x + r1.w > r2.x && r1.y < r2.y + r2.h && r1.y + r1.h > r2.y;
        }

        function die() {
            player.reset(50, 300);
            coinsCollected = 0;
            coinDisplay.innerText = 0;
            map.coins.forEach(c => c.collected = false);
        }

        function nextLevel() {
            if (currentLevel < 10) {
                currentLevel++;
                levelDisplay.innerText = currentLevel;
                coinsCollected = 0;
                coinDisplay.innerText = 0;
                map = getLevelData(currentLevel);
                player.reset(50, 300);
                alert(`축하합니다! 레벨 ${currentLevel}로 이동합니다.`);
            } else {
                gameState = "SUCCESS";
                alert("🎉 모든 모험을 마쳤습니다! 당신은 위대한 병아리입니다! 🎉");
            }
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // 배경 구름 느낌 (간단)
            ctx.fillStyle = "rgba(255,255,255,0.3)";
            ctx.beginPath(); ctx.arc(100, 100, 40, 0, Math.PI*2); ctx.fill();
            ctx.beginPath(); ctx.arc(600, 80, 30, 0, Math.PI*2); ctx.fill();

            // 발판 그림 (도트 스타일)
            ctx.fillStyle = "#5d4037";
            map.platforms.forEach(p => {
                ctx.fillRect(p.x, p.y, p.w, p.h);
                ctx.fillStyle = "#8bc34a"; // 풀 상단
                ctx.fillRect(p.x, p.y, p.w, 5);
                ctx.fillStyle = "#5d4037";
            });

            // 가시 (🔺)
            ctx.fillStyle = "#757575";
            map.obstacles.forEach(o => {
                ctx.beginPath();
                ctx.moveTo(o.x, o.y + o.h);
                ctx.lineTo(o.x + o.w/2, o.y);
                ctx.lineTo(o.x + o.w, o.y + o.h);
                ctx.fill();
            });

            // 농부 (👨‍🌾)
            map.enemies.forEach(e => {
                ctx.font = "30px Arial";
                ctx.fillText("👨‍🌾", e.x, e.y + 30);
            });

            // 코인 (💰)
            map.coins.forEach(c => {
                if (!c.collected) {
                    ctx.font = "20px Arial";
                    ctx.fillText("💰", c.x, c.y + 20);
                }
            });

            // 병아리 (🐥)
            ctx.save();
            if (player.facing === -1) { // 왼쪽 볼 때 반전
                ctx.translate(player.x + player.w, player.y);
                ctx.scale(-1, 1);
                ctx.font = "30px Arial";
                ctx.fillText("🐥", 0, 25);
            } else {
                ctx.font = "30px Arial";
                ctx.fillText("🐥", player.x, player.y + 25);
            }
            ctx.restore();

            // 벽타기 게이지 표시 (벽에 붙었을 때만)
            if (player.isClimbing) {
                ctx.fillStyle = "red";
                const gaugeWidth = (1 - (player.climbTimer / CLIMB_TIME_LIMIT)) * 30;
                ctx.fillRect(player.x, player.y - 10, gaugeWidth, 5);
            }
        }

        function gameLoop() {
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }

        gameLoop();
    </script>
</body>
</html>
