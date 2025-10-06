
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>中秋快乐</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: linear-gradient(135deg, #667eea, #764ba2);
            color: white;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            padding: 20px;
            text-align: center;
        }
        .container {
            max-width: 100%;
            width: 100%;
        }
        .moon {
            font-size: 80px;
            margin: 20px 0;
            animation: bounce 2s infinite;
        }
        .title {
            font-size: 28px;
            font-weight: bold;
            margin: 15px 0;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
        }
        .blessing {
            font-size: 20px;
            margin: 12px 0;
            opacity: 0;
            animation: fadeIn 1s forwards;
        }
        .emoji-row {
            font-size: 24px;
            margin: 20px 0;
            display: flex;
            justify-content: center;
            gap: 15px;
        }
        .footer {
            margin-top: 30px;
            font-size: 14px;
            opacity: 0.8;
        }
        @keyframes bounce {
            0%, 100% { transform: scale(1); }
            50% { transform: scale(1.1); }
        }
        @keyframes fadeIn {
            to { opacity: 1; }
        }
        .touch-area {
            padding: 20px;
            margin: 15px;
            background: rgba(255,255,255,0.1);
            border-radius: 20px;
            backdrop-filter: blur(10px);
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="touch-area">
            <div class="moon">🌕</div>
            <div class="title" id="mainTitle">中秋快乐</div>

            <div class="blessing" style="animation-delay: 0.5s" id="blessing1">月圆人团圆</div>
            <div class="blessing" style="animation-delay: 1s" id="blessing2">事事都圆满</div>
            <div class="blessing" style="animation-delay: 1.5s" id="blessing3">幸福永相伴</div>

            <div class="emoji-row">
                <span>🥮</span>
                <span>🐇</span>
                <span>🏮</span>
                <span>✨</span>
            </div>
        </div>

        <div class="footer">
            触摸屏幕有惊喜 | 中秋祝福
        </div>
    </div>

    <script>
        // 祝福语数组
        const blessings = [
            ["中秋快乐", "月圆人团圆", "幸福永相伴"],
            ["节日愉快", "阖家幸福", "万事如意"], 
            ["月饼甜甜", "月亮圆圆", "祝福满满"],
            ["愿您安康", "祝您幸福", "盼您快乐"]
        ];

        let currentSet = 0;

        // 点击切换祝福语
        document.body.addEventListener('click', function() {
            currentSet = (currentSet + 1) % blessings.length;

            document.getElementById('mainTitle').textContent = blessings[currentSet][0];
            document.getElementById('blessing1').textContent = blessings[currentSet][1];
            document.getElementById('blessing2').textContent = blessings[currentSet][2];

            // 添加点击效果
            const sparkles = ['✨', '🌟', '💫', '🎊'];
            const sparkle = document.createElement('div');
            sparkle.textContent = sparkles[Math.floor(Math.random() * sparkles.length)];
            sparkle.style.position = 'fixed';
            sparkle.style.left = Math.random() * 80 + 10 + '%';
            sparkle.style.top = Math.random() * 80 + 10 + '%';
            sparkle.style.fontSize = '30px';
            sparkle.style.animation = 'fadeIn 0.5s ease-out forwards';
            document.body.appendChild(sparkle);

            setTimeout(() => {
                sparkle.remove();
            }, 1000);
        });

        // 自动轮播
        setInterval(() => {
            document.body.click();
        }, 5000);

        // 触摸事件支持
        document.body.addEventListener('touchstart', function(e) {
            document.body.click();
        });
    </script>
</body>
</html>
