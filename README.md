<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Молния Маккуин: Дрифт в Радиатор-Спрингс</title>
    <style>
        * {
            user-select: none;
            -webkit-tap-highlight-color: transparent;
        }

        body {
            background: linear-gradient(135deg, #0f2b1f 0%, #1a3a2a 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Segoe UI', 'Impact', 'Arial Narrow', sans-serif;
            margin: 0;
            padding: 20px;
        }

        .game-wrapper {
            background: #2c2118;
            border-radius: 60px;
            padding: 20px;
            box-shadow: 0 20px 35px rgba(0,0,0,0.6), inset 0 1px 3px rgba(255,215,140,0.3);
            border: 2px solid #e0872e;
        }

        .game-panel {
            background: #3d2a1c;
            border-radius: 40px;
            padding: 15px;
            box-shadow: inset 0 0 12px #1a120a;
        }

        canvas {
            display: block;
            margin: 0 auto;
            border-radius: 28px;
            box-shadow: 0 0 0 5px #f5b642, 0 12px 28px black;
            cursor: pointer;
            background: #5f7b4a;
        }

        .info-panel {
            display: flex;
            justify-content: space-between;
            align-items: baseline;
            margin-top: 20px;
            gap: 15px;
            flex-wrap: wrap;
            justify-content: center;
        }

        .score-card {
            background: #000000aa;
            backdrop-filter: blur(4px);
            border-radius: 50px;
            padding: 6px 22px;
            font-weight: bold;
            color: #ffdd99;
            font-size: 1.6rem;
            font-family: monospace;
            border-left: 4px solid #f5a623;
            border-right: 4px solid #f5a623;
        }

        .score-card span {
            color: #ffae42;
            font-size: 1.9rem;
            margin-right: 8px;
        }

        .drift-meter {
            background: #1e1510;
            border-radius: 40px;
            padding: 5px 18px;
            font-weight: bold;
            color: #ffaa66;
            font-size: 1.2rem;
            font-family: monospace;
        }

        .message-area {
            background: #030e0bdd;
            backdrop-filter: blur(12px);
            border-radius: 36px;
            padding: 6px 20px;
            font-weight: bold;
            font-size: 1rem;
            color: #f7d44a;
            text-align: center;
        }

        button {
            background: #ce5a2a;
            border: none;
            font-weight: bold;
            font-size: 1.2rem;
            font-family: inherit;
            padding: 8px 30px;
            border-radius: 60px;
            color: white;
            text-shadow: 0 1px 0 #6e2a0a;
            box-shadow: 0 5px 0 #81341c;
            cursor: pointer;
            transition: 0.05s linear;
            letter-spacing: 1px;
        }

        button:active {
            transform: translateY(3px);
            box-shadow: 0 2px 0 #81341c;
        }

        .control-hint {
            background: #b95f1f40;
            border-radius: 30px;
            padding: 5px 12px;
            font-size: 0.8rem;
            font-weight: bold;
            color: #ffd58c;
            text-align: center;
            margin-top: 8px;
        }

        @keyframes shake {
            0% { transform: translateX(0px);}
            25% { transform: translateX(4px);}
            75% { transform: translateX(-4px);}
            100% { transform: translateX(0px);}
        }

        .crash-effect {
            animation: shake 0.1s ease-in-out 3;
        }

        footer {
            font-size: 0.7rem;
            text-align: center;
            margin-top: 12px;
            color: #c6a15b;
        }
    </style>
</head>
<body>
<div>
    <div class="game-wrapper">
        <div class="game-panel">
            <canvas id="gameCanvas" width="800" height="450" style="width:100%; height:auto; max-width:800px; aspect-ratio:800/450"></canvas>

            <div class="info-panel">
                <div class="score-card"><span>🏆</span> ОЧКИ: <span id="scoreValue">0</span></div>
                <div class="drift-meter">🌀 ДРИФТ: <span id="driftCombo">0</span> сек</div>
                <div class="message-area" id="statusMsg">▶ НАЖМИ A / D или СТРЕЛКИ ◀</div>
            </div>
            <div style="display: flex; justify-content: center; gap: 20px; margin-top: 12px;">
                <button id="resetBtn">ЗАНОВО 🏁</button>
            </div>
            <div class="control-hint">
                🏎️ A / ← ЛЕВО | D / → ПРАВО | Удерживай поворот для ДРИФТА! Обходи кактусы 🌵
            </div>
            <footer>⭐ Радиатор-Спрингс: держись на серпантине и избегай кактусов! ⭐</footer>
        </div>
    </div>
</div>

<script>
    (function(){
        // ----- ИГРА: ДРИФТ В РАДИАТОР-СПРИНГС -----
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        canvas.width = 800;
        canvas.height = 450;
        
        // ----- ИГРОВЫЕ ПАРАМЕТРЫ -----
        let score = 0;              // очки за выживание и дрифт
        let driftTimer = 0;        // сколько секунд непрерывно дрифтуем (комбо)
        let isDrifting = false;     // зажат ли поворот
        let driftDirection = 0;     // -1 левый дрифт, 1 правый дрифт
        let gameRunning = true;
        
        // машинка
        let carX = canvas.width/2;
        const carWidth = 48;
        const carHeight = 36;
        let laneSpeed = 0;          // эффект скольжения/инерции
        let roadCurve = 0;          // визуальный изгиб дороги
        
        // препятствия (кактусы)
        let obstacles = [];
        let frameCounter = 0;
        let spawnGap = 45;           // через сколько кадров spawn
        let baseSpeed = 3.2;         // скорость движения препятствий
        
        // эффекты
        let invincibleFrames = 0;     // после столкновения неуязвимость
        let shakeAmount = 0;
        let messageTimeout = null;
        
        // рекорд
        let highScore = 0;
        
        // управление
        const keys = {
            ArrowLeft: false,
            ArrowRight: false,
            KeyA: false,
            KeyD: false
        };
        
        // ----- Вспомогательные функции -----
        function setStatusMessage(msg, isError = false) {
            const msgDiv = document.getElementById('statusMsg');
            if(messageTimeout) clearTimeout(messageTimeout);
            if(msgDiv) {
                msgDiv.style.color = isError ? "#ffaa88" : "#f7e05e";
                msgDiv.innerText = msg;
            }
            messageTimeout = setTimeout(() => {
                if(document.getElementById('statusMsg')) {
                    document.getElementById('statusMsg').style.color = "#f7d44a";
                    document.getElementById('statusMsg').innerText = "🌀 ДРИФТУЙ НА ПОВОРОТАХ! ОБХОДИ КАКТУСЫ 🌵";
                }
            }, 1800);
        }
        
        function updateUI() {
            document.getElementById('scoreValue').innerText = Math.floor(score);
            document.getElementById('driftCombo').innerText = driftTimer.toFixed(1);
            if(score > highScore) {
                highScore = Math.floor(score);
            }
        }
        
        // столкновение
        function crash() {
            if(invincibleFrames > 0) return false;
            // сброс дрифт комбо
            driftTimer = 0;
            isDrifting = false;
            setStatusMessage("💥 ВРЕЗАЛСЯ В КАКТУС! СТОП-ДРИФТ! 💥", true);
            // штраф 50 очков, но не ниже 0
            score = Math.max(0, score - 40);
            updateUI();
            invincibleFrames = 35;   // около 0.6 секунды неуязвимости
            shakeAmount = 12;
            // тряска канваса
            if(canvas.parentElement) {
                canvas.parentElement.classList.add('crash-effect');
                setTimeout(() => {
                    if(canvas.parentElement) canvas.parentElement.classList.remove('crash-effect');
                }, 300);
            }
            // вибрация
            if(navigator.vibrate) navigator.vibrate(120);
            return true;
        }
        
        // добавление кактуса
        function spawnObstacle() {
            let randX = 120 + Math.random() * (canvas.width - carWidth - 120);
            // не слишком близко к краям
            randX = Math.min(canvas.width - carWidth - 15, Math.max(15, randX));
            obstacles.push({
                x: randX,
                y: -30,
                width: 28,
                height: 42,
                type: 'cactus'
            });
        }
        
        // обновление логики
        function updateGame() {
            if(!gameRunning) return;
            
            // 1. управление и дрифт
            let leftPressed = keys.ArrowLeft || keys.KeyA;
            let rightPressed = keys.ArrowRight || keys.KeyD;
            let move = 0;
            if(leftPressed) move = -1;
            if(rightPressed) move = 1;
            
            // инерция + скорость поворота
            laneSpeed = laneSpeed * 0.92 + move * 4.2;
            carX += laneSpeed;
            // границы дороги (отступы)
            carX = Math.min(canvas.width - carWidth - 12, Math.max(12, carX));
            
            // ЛОГИКА ДРИФТА: дрифт активируется если зажат поворот и есть боковое движение > 1.5
            const isTurning = (move !== 0) && Math.abs(laneSpeed) > 1.8;
            if(isTurning && !isDrifting) {
                isDrifting = true;
                driftDirection = (laneSpeed > 0) ? 1 : -1;
                setStatusMessage("🔥 ДРИФТ! Удерживай поворот 🔥");
            } 
            else if(!isTurning && isDrifting) {
                isDrifting = false;
                driftDirection = 0;
                if(driftTimer > 0.5) setStatusMessage("✅ Дрифт завершён + очки!");
            }
            
            // начисление очков за дрифт (время)
            if(isDrifting) {
                let driftPoints = 1.3 * (1 + Math.min(driftTimer * 0.1, 1.5));
                score += driftPoints * 0.25;
                driftTimer += 0.016;   // примерно кадр 60fps
                if(driftTimer > 15) driftTimer = 15;
            } else {
                if(driftTimer > 0) {
                    // бонус за серию дрифта (добавим к счёту маленький бонус)
                    let bonus = Math.floor(driftTimer * 3);
                    if(bonus > 0 && driftTimer > 0.3) {
                        score += bonus;
                        setStatusMessage(`🏁 Бонус дрифта +${bonus}! 🏁`);
                    }
                    driftTimer = 0;
                }
            }
            
            // 2. Спавн кактусов
            frameCounter++;
            let dynamicSpawn = Math.max(28, spawnGap - Math.floor(score / 450));
            if(frameCounter >= dynamicSpawn) {
                frameCounter = 0;
                spawnObstacle();
                // иногда двойной кактус при большом счете
                if(score > 600 && Math.random() < 0.25) {
                    spawnObstacle();
                }
            }
            
            // 3. обновление препятствий
            let currentSpeed = baseSpeed + Math.min(score / 800, 2.5);
            for(let i=0; i<obstacles.length; i++) {
                obstacles[i].y += currentSpeed;
            }
            // удалить вышедшие за экран + начисление очков за пропуск
            obstacles = obstacles.filter(obs => {
                if(obs.y > canvas.height + 80) {
                    score += 12;   // бонус за успешный обгон
                    updateUI();
                    return false;
                }
                return true;
            });
            
            // 4. проверка столкновения
            for(let i=0; i<obstacles.length; i++) {
                const obs = obstacles[i];
                if(invincibleFrames <= 0) {
                    if(carX < obs.x + obs.width &&
                        carX + carWidth > obs.x &&
                        canvas.height - 70 < obs.y + obs.height &&
                        canvas.height - 35 > obs.y) {
                        crash();
                        // сдвинем машину для реализма
                        laneSpeed = -laneSpeed * 0.6;
                        carX += (laneSpeed * 2);
                        carX = Math.min(canvas.width - carWidth - 12, Math.max(12, carX));
                        break;
                    }
                }
            }
            
            // уменьшаем неуязвимость и эффект тряски
            if(invincibleFrames > 0) invincibleFrames--;
            if(shakeAmount > 0) shakeAmount -= 0.6;
            else shakeAmount = 0;
            
            // добавим очки за выживание
            score += 0.18;
            updateUI();
        }
        
        // ---- БОЛЬШАЯ ОТРИСОВКА ----
        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // горный фон + небо
            const gradSky = ctx.createLinearGradient(0,0,0,250);
            gradSky.addColorStop(0,"#b77c48");
            gradSky.addColorStop(1,"#e0b072");
            ctx.fillStyle = gradSky;
            ctx.fillRect(0,0,canvas.width,200);
            ctx.fillStyle = "#7d5535";
            for(let i=0;i<8;i++) {
                ctx.beginPath();
                ctx.moveTo(i*120,150);
                ctx.lineTo(i*120+70,70);
                ctx.lineTo(i*120-70,70);
                ctx.fill();
            }
            // песок/дорога
            ctx.fillStyle = "#ba874a";
            ctx.fillRect(0,canvas.height-80,canvas.width,80);
            ctx.fillStyle = "#7c5a36";
            for(let i=0;i<12;i++) {
                ctx.fillRect(i*70+ (Date.now()*0.2)%70 , canvas.height-78, 35, 6);
            }
            // асфальт
            ctx.fillStyle = "#2e2b23";
            ctx.fillRect(0,canvas.height-68,canvas.width,55);
            ctx.fillStyle = "#f5cb8b";
            ctx.beginPath();
            ctx.setLineDash([20,35]);
            ctx.lineWidth = 4;
            ctx.moveTo(0,canvas.height-42);
            ctx.lineTo(canvas.width,canvas.height-42);
            ctx.stroke();
            ctx.setLineDash([]);
            
            // отрисовка кактусов
            for(let obs of obstacles) {
                // рисуем кактус
                ctx.fillStyle = "#2f6b2f";
                ctx.shadowBlur = 3;
                ctx.beginPath();
                ctx.roundRect(obs.x, obs.y, obs.width, obs.height, 8);
                ctx.fill();
                ctx.fillStyle = "#1e4a1e";
                ctx.beginPath();
                ctx.roundRect(obs.x+6, obs.y-12, 6, 18, 4);
                ctx.fill();
                ctx.beginPath();
                ctx.roundRect(obs.x+16, obs.y-8, 6, 16, 4);
                ctx.fill();
                ctx.fillStyle = "#c9ae34";
                for(let s=0;s<3;s++) {
                    ctx.beginPath();
                    ctx.moveTo(obs.x+5+s*6, obs.y-4);
                    ctx.lineTo(obs.x+2+s*6, obs.y-12);
                    ctx.lineTo(obs.x+8+s*6, obs.y-12);
                    ctx.fill();
                }
            }
            ctx.shadowBlur = 0;
            
            // МАШИНА МОЛНИЯ МАККУИН
            let carDrawX = carX;
            let carDrawY = canvas.height - 58;
            // эффект дрифта - наклон
            let tilt = 0;
            if(isDrifting && Math.abs(laneSpeed) > 1) {
                tilt = laneSpeed * 0.12;
                if(tilt > 10) tilt = 10;
                if(tilt < -10) tilt = -10;
                // дым из-под колес
                for(let i=0;i<3;i++) {
                    ctx.fillStyle = `rgba(180,140,100,${0.5-Math.random()*0.3})`;
                    ctx.beginPath();
                    ctx.ellipse(carDrawX-10+(i*10)+ (Math.random()*8), carDrawY+25, 7, 4, 0, 0, Math.PI*2);
                    ctx.fill();
                }
            }
            
            ctx.save();
            ctx.translate(carDrawX+carWidth/2, carDrawY+carHeight/2);
            ctx.rotate(tilt * 0.05);
            ctx.translate(-(carDrawX+carWidth/2), -(carDrawY+carHeight/2));
            // кузов
            ctx.fillStyle = "#e0452c";
            ctx.beginPath();
            ctx.roundRect(carDrawX, carDrawY, carWidth, carHeight, 12);
            ctx.fill();
            ctx.fillStyle = "#b1311a";
            ctx.beginPath();
            ctx.roundRect(carDrawX+8, carDrawY-5, 32, 10, 5);
            ctx.fill();
            // молния
            ctx.fillStyle = "#fdcc4a";
            ctx.beginPath();
            ctx.moveTo(carDrawX+30, carDrawY+12);
            ctx.lineTo(carDrawX+42, carDrawY+6);
            ctx.lineTo(carDrawX+36, carDrawY+18);
            ctx.lineTo(carDrawX+46, carDrawY+18);
            ctx.lineTo(carDrawX+32, carDrawY+30);
            ctx.lineTo(carDrawX+38, carDrawY+20);
            ctx.lineTo(carDrawX+28, carDrawY+20);
            ctx.fill();
            // номер 95
            ctx.font = "bold 18px 'Impact'";
            ctx.fillStyle = "#fff0b5";
            ctx.fillText("95", carDrawX+18, carDrawY+27);
            // колеса
            ctx.fillStyle = "#1c1c1c";
            ctx.beginPath();
            ctx.ellipse(carDrawX+8, carDrawY+carHeight-6, 9, 9, 0, 0, Math.PI*2);
            ctx.fill();
            ctx.beginPath();
            ctx.ellipse(carDrawX+carWidth-12, carDrawY+carHeight-6, 9, 9, 0, 0, Math.PI*2);
            ctx.fill();
            ctx.fillStyle = "#888";
            ctx.beginPath();
            ctx.ellipse(carDrawX+8, carDrawY+carHeight-6, 5, 5, 0, 0, Math.PI*2);
            ctx.fill();
            ctx.beginPath();
            ctx.ellipse(carDrawX+carWidth-12, carDrawY+carHeight-6, 5, 5, 0, 0, Math.PI*2);
            ctx.fill();
            
            // глаза Маккуина
            ctx.fillStyle = "white";
            ctx.beginPath();
            ctx.arc(carDrawX+12, carDrawY+12, 6, 0, Math.PI*2);
            ctx.fill();
            ctx.beginPath();
            ctx.arc(carDrawX+38, carDrawY+12, 6, 0, Math.PI*2);
            ctx.fill();
            ctx.fillStyle = "#232323";
            ctx.beginPath();
            ctx.arc(carDrawX+10 + (isDrifting?1:0), carDrawY+11, 3, 0, Math.PI*2);
            ctx.fill();
            ctx.beginPath();
            ctx.arc(carDrawX+36 + (isDrifting?1:0), carDrawY+11, 3, 0, Math.PI*2);
            ctx.fill();
            ctx.restore();
            
            // неуязвимость (мерцание)
            if(invincibleFrames > 0 && (Math.floor(Date.now()/50) % 3 === 0)) {
                ctx.globalAlpha = 0.6;
                ctx.fillStyle = "#ffffaa";
                ctx.beginPath();
                ctx.roundRect(carDrawX-4, carDrawY-4, carWidth+8, carHeight+8, 12);
                ctx.fill();
                ctx.globalAlpha = 1;
            }
            
            // индикатор дрифта
            if(isDrifting) {
                ctx.font = "bold 18px monospace";
                ctx.fillStyle = "#ffb347";
                ctx.shadowBlur = 6;
                ctx.fillText("🌀 DRIFT! +" + driftTimer.toFixed(1) + "s", carDrawX-10, carDrawY-20);
                ctx.fillStyle = "#f2851c";
                ctx.fillText("🌀 DRIFT! +" + driftTimer.toFixed(1) + "s", carDrawX-12, carDrawY-22);
            }
            
            // тряска камеры при столкновении
            if(shakeAmount > 0) {
                ctx.translate((Math.random() - 0.5)*shakeAmount, (Math.random() - 0.5)*shakeAmount*0.5);
            }
            ctx.shadowBlur = 0;
            
            // счёт рекорда
            ctx.font = "bold 12px monospace";
            ctx.fillStyle = "#fad98b";
            ctx.fillText("⭐ РЕКОРД: " + Math.floor(highScore), canvas.width-120, 28);
            ctx.fillStyle = "#eaac6f";
            ctx.fillText("🌵 КАКТУСЫ НА ДОРОГЕ!", 14, 32);
        }
        
        // цикл анимации
        let lastTime = 0;
        function gameLoop() {
            updateGame();
            draw();
            requestAnimationFrame(gameLoop);
        }
        
        // инициализация управлений
        function handleKeyDown(e) {
            let code = e.code;
            if(keys.hasOwnProperty(code)) {
                keys[code] = true;
                e.preventDefault();
            }
            // дополнительно R для рестарта
            if(code === 'KeyR') {
                fullReset();
                e.preventDefault();
            }
        }
        
        function handleKeyUp(e) {
            let code = e.code;
            if(keys.hasOwnProperty(code)) {
                keys[code] = false;
                e.preventDefault();
            }
        }
        
        function fullReset() {
            score = 0;
            driftTimer = 0;
            isDrifting = false;
            obstacles = [];
            carX = canvas.width/2;
            laneSpeed = 0;
            invincibleFrames = 0;
            frameCounter = 0;
            setStatusMessage("🏎️ НОВАЯ ГОНКА! ДЕРЖИ ДРИФТ! 🏎️");
            updateUI();
        }
        
        window.addEventListener('keydown', handleKeyDown);
        window.addEventListener('keyup', handleKeyUp);
        const resetButton = document.getElementById('resetBtn');
        resetButton.addEventListener('click', () => fullReset());
        
        // старт
        fullReset();
        setStatusMessage("🌀 ЖМИ A/D или СТРЕЛКИ для дрифта и обхода!");
        updateUI();
        gameLoop();
        
        // Полифилл roundRect
        if (!CanvasRenderingContext2D.prototype.roundRect) {
            CanvasRenderingContext2D.prototype.roundRect = function(x, y, w, h, r) {
                if (w < 2 * r) r = w / 2;
                if (h < 2 * r) r = h / 2;
                this.moveTo(x+r, y);
                this.lineTo(x+w-r, y);
                this.quadraticCurveTo(x+w, y, x+w, y+r);
                this.lineTo(x+w, y+h-r);
                this.quadraticCurveTo(x+w, y+h, x+w-r, y+h);
                this.lineTo(x+r, y+h);
                this.quadraticCurveTo(x, y+h, x, y+h-r);
                this.lineTo(x, y+r);
                this.quadraticCurveTo(x, y, x+r, y);
                return this;
            };
        }
    })();
</script>
</body>
</html>
