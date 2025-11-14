<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>لعبة قتال بسيطة</title>
    
    <style>
        body {
            font-family: 'Arial', sans-serif;
            background-color: #333;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            direction: rtl; 
            color: white;
        }

        #gameCanvas {
            background-color: #222;
            border: 5px solid #FF0000;
            box-shadow: 0 0 15px rgba(255, 0, 0, 0.7);
        }

        #instructions {
            margin-top: 15px;
            text-align: center;
            display: flex;
            gap: 50px;
        }

        .controls {
            border: 1px solid #FFEB3B;
            padding: 10px;
            border-radius: 5px;
            text-align: right;
        }
        .controls h3 {
            color: #FFEB3B;
        }
    </style>
</head>
<body>
    <div class="game-container">
        <canvas id="gameCanvas" width="800" height="400"></canvas>
        
        <div id="instructions">
            <div class="controls">
                <h3>اللاعب 1 (الأزرق)</h3>
                <p>الحركة: **W, A, D**</p>
                <p>اللكم: **E**</p>
            </div>
            <div class="controls">
                <h3>اللاعب 2 (الأحمر)</h3>
                <p>الحركة: **الأسهم (⬆️ ⬅️ ➡️)**</p>
                <p>اللكم: **M**</p>
            </div>
        </div>
    </div>
    
    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const canvasWidth = canvas.width;
        const canvasHeight = canvas.height;
        const groundY = canvasHeight - 50;
        
        let gameOver = false;

        // === دالة بناء اللاعبين ===
        function Player(x, keys) {
            this.x = x;
            this.y = groundY - 50;
            this.width = 40;
            this.height = 80;
            this.color = keys.color;
            this.health = 100;
            this.speed = 5;
            this.vx = 0;
            this.vy = 0;
            this.isJumping = false;
            this.isAttacking = false;
            this.attackFrame = 0; // لضبط مدة الهجوم
            this.controls = keys; // A, D, W, E...
        }

        const gravity = 0.5;
        const jumpStrength = 12;
        const attackDuration = 10; // عدد الإطارات للهجوم

        // === إنشاء اللاعبين ===
        const player1 = new Player(100, { 
            moveLeft: 'a', moveRight: 'd', jump: 'w', attack: 'e', color: 'blue'
        });

        const player2 = new Player(canvasWidth - 100 - 40, { 
            moveLeft: 'ArrowLeft', moveRight: 'ArrowRight', jump: 'ArrowUp', attack: 'm', color: 'red'
        });
        
        let keysPressed = {}; // لتسجيل المفاتيح

        // === الدوال الأساسية ===

        function drawRect(x, y, w, h, color) {
            ctx.fillStyle = color;
            ctx.fillRect(x, y, w, h);
        }

        function drawPlayer(p) {
            // رسم شريط الصحة (Health Bar)
            drawRect(p.x, p.y - 20, p.width, 10, 'grey');
            drawRect(p.x, p.y - 20, p.width * (p.health / 100), 10, 'lime');

            // رسم الجسم
            drawRect(p.x, p.y, p.width, p.height, p.color);

            // رسم مربع الهجوم (Attack Hitbox)
            if (p.isAttacking) {
                // تحديد الاتجاه الذي يواجهه اللاعب للهجوم فيه
                const attackOffset = p === player1 ? p.width : -p.width;
                drawRect(p.x + attackOffset, p.y + 20, p.width, 20, 'rgba(255, 255, 0, 0.5)');
            }
        }
        
        function updateMovement(p) {
            // 1. تطبيق الجاذبية
            if (p.y < groundY - p.height) {
                p.vy += gravity;
                p.y += p.vy;
            } else {
                p.y = groundY - p.height;
                p.vy = 0;
                p.isJumping = false;
            }

            // 2. الحركة الأفقية
            p.vx = 0;
            if (keysPressed[p.controls.moveRight]) {
                p.vx = p.speed;
            }
            if (keysPressed[p.controls.moveLeft]) {
                p.vx = -p.speed;
            }
            p.x += p.vx;

            // 3. القفز
            if (keysPressed[p.controls.jump] && !p.isJumping) {
                p.vy = -jumpStrength;
                p.isJumping = true;
            }

            // 4. منع الخروج من الحدود
            if (p.x < 0) p.x = 0;
            if (p.x + p.width > canvasWidth) p.x = canvasWidth - p.width;

            // 5. تحديث حالة الهجوم
            if (p.isAttacking) {
                p.attackFrame++;
                if (p.attackFrame >= attackDuration) {
                    p.isAttacking = false;
                    p.attackFrame = 0;
                }
            }
        }
        
        // دالة اكتشاف الاصطدام بين مربع الهجوم واللاعب الآخر
        function checkAttack(attacker, target) {
            if (attacker.isAttacking) {
                // تحديد مربع الهجوم
                const attackOffset = attacker === player1 ? attacker.width : -attacker.width;
                const attackX = attacker.x + attackOffset;
                const attackY = attacker.y + 20;
                const attackW = attacker.width;
                const attackH = 20;

                // التحقق من التداخل
                if (
                    attackX < target.x + target.width &&
                    attackX + attackW > target.x &&
                    attackY < target.y + target.height &&
                    attackY + attackH > target.y
                ) {
                    // تم الضرب!
                    target.health -= 10;
                    attacker.isAttacking = false; // إيقاف الهجوم بعد الضربة
                    
                    if (target.health <= 0) {
                        target.health = 0;
                        gameOver = true;
                    }
                }
            }
        }


        // دالة التحديث الرئيسية
        function update() {
            if (gameOver) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.8)';
                ctx.fillRect(0, 0, canvasWidth, canvasHeight);
                ctx.fillStyle = 'white';
                ctx.font = '40px Arial';
                ctx.textAlign = 'center';
                const winner = (player1.health > 0) ? "اللاعب 1 (الأزرق)" : "اللاعب 2 (الأحمر)";
                ctx.fillText(`انتهت اللعبة! الفائز هو ${winner}`, canvasWidth / 2, canvasHeight / 2);
                ctx.font = '20px Arial';
                ctx.fillText("اضغط F5 للبدء من جديد", canvasWidth / 2, canvasHeight / 2 + 50);
                return;
            }

            // 1. مسح الشاشة
            ctx.clearRect(0, 0, canvasWidth, canvasHeight);
            
            // 2. رسم الأرضية
            drawRect(0, groundY, canvasWidth, 50, 'grey');
            
            // 3. تحديث الحركة
            updateMovement(player1);
            updateMovement(player2);
            
            // 4. فحص الهجمات
            checkAttack(player1, player2);
            checkAttack(player2, player1);
            
            // 5. رسم اللاعبين
            drawPlayer(player1);
            drawPlayer(player2);

            requestAnimationFrame(update);
        }

        // === التحكم بلوحة المفاتيح ===
        document.addEventListener('keydown', (e) => {
            keysPressed[e.key.toLowerCase()] = true;
            
            // التعامل مع ضغطة الهجوم لمرة واحدة
            if (e.key.toLowerCase() === player1.controls.attack && !player1.isAttacking) {
                player1.isAttacking = true;
                player1.attackFrame = 0;
            }
            if (e.key.toLowerCase() === player2.controls.attack && !player2.isAttacking) {
                player2.isAttacking = true;
                player2.attackFrame = 0;
            }
        });

        document.addEventListener('keyup', (e) => {
            keysPressed[e.key.toLowerCase()] = false;
        });

        // === بدء اللعبة ===
        requestAnimationFrame(update);
    </script>
</body>
</html>
