<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cookie Clicker - Persistent Edition</title>
    <style>
        body { font-family: 'Tahoma', sans-serif; background: #2c1e14; color: white; margin: 0; height: 100vh; overflow: hidden; user-select: none; }
        #game-container { display: flex; width: 100%; height: 100%; }
        .click-area { flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; background: radial-gradient(circle, #4e342e 0%, #2c1e14 100%); position: relative; }
        .stats-panel { background: rgba(0,0,0,0.5); padding: 20px; border-radius: 15px; text-align: center; margin-bottom: 20px; min-width: 250px; }
        .stats-panel h2 { margin: 0; color: #ffb74d; font-size: 2.2rem; }
        #cookie-btn { font-size: 160px; background: none; border: none; cursor: pointer; outline: none; transition: transform 0.05s; z-index: 2; }
        #cookie-btn:active { transform: scale(0.95); }
        .shop-area { width: 350px; background: #3e2723; border-left: 5px solid #1a0f0d; display: flex; flex-direction: column; }
        .shop-header { padding: 15px; text-align: center; background: #2a1a17; font-weight: bold; border-bottom: 2px solid #1a0f0d; }
        .building-list { overflow-y: auto; flex-grow: 1; padding: 10px; }
        .building { display: none; background: #5d4037; margin-bottom: 10px; padding: 12px; border-radius: 8px; justify-content: space-between; align-items: center; cursor: pointer; border: 2px solid #795548; }
        .building.visible { display: flex; }
        .building.disabled { opacity: 0.5; filter: grayscale(0.8); cursor: not-allowed; }
        .footer { padding: 10px; background: #1a0f0d; display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
        button { cursor: pointer; border: none; border-radius: 5px; padding: 10px; font-weight: bold; color: white; }
        .save-btn { background: #388e3c; }
        .reset-btn { background: #c0392b; }
        #save-popup { position: absolute; top: 20px; right: 20px; background: rgba(56, 142, 60, 0.9); padding: 10px 20px; border-radius: 5px; display: none; animation: fadeout 2s forwards; }
        @keyframes fadeout { 0% { opacity: 1; } 80% { opacity: 1; } 100% { opacity: 0; } }
    </style>
</head>
<body>
    <div id="game-container">
        <div class="click-area">
            <div id="save-popup">Game Saved!</div>
            <div class="stats-panel">
                <h2 id="cookie-display">0 Cookies</h2>
                <div id="cps-display">per second: 0.0</div>
            </div>
            <button id="cookie-btn">🍪</button>
        </div>
        <div class="shop-area">
            <div class="shop-header">STORE</div>
            <div id="shop-list" class="building-list"></div>
            <div class="footer">
                <button class="save-btn" onclick="manualSave()">SAVE</button>
                <button class="reset-btn" onclick="resetGame()">RESET</button>
            </div>
        </div>
    </div>

    <script>
        const buildings = [
            { id: 'cursor', name: 'Cursor', baseCost: 15, cps: 0.1 },
            { id: 'grandma', name: 'Grandma', baseCost: 100, cps: 1 },
            { id: 'farm', name: 'Farm', baseCost: 1100, cps: 8 },
            { id: 'mine', name: 'Mine', baseCost: 12000, cps: 47 }
        ];

        let state = { cookies: 0, owned: {} };
        buildings.forEach(b => state.owned[b.id] = 0);

        function updateUI() {
            document.getElementById('cookie-display').innerText = Math.floor(state.cookies).toLocaleString() + " Cookies";
            let totalCPS = 0;
            buildings.forEach(b => {
                const count = state.owned[b.id] || 0;
                const cost = Math.floor(b.baseCost * Math.pow(1.15, count));
                totalCPS += (count * b.cps);
                const row = document.getElementById(`row-${b.id}`);
                if (row) {
                    row.querySelector('.cost').innerText = `Cost: ${cost.toLocaleString()}`;
                    row.querySelector('.count').innerText = count;
                    row.classList.toggle('visible', count > 0 || state.cookies >= b.baseCost * 0.8);
                    row.classList.toggle('disabled', state.cookies < cost);
                }
            });
            document.getElementById('cps-display').innerText = `per second: ${totalCPS.toFixed(1)}`;
        }

        function saveGame() {
            localStorage.setItem('cookie_save_v1', JSON.stringify(state));
            console.log("Saved to storage.");
        }

        function manualSave() {
            saveGame();
            const pop = document.getElementById('save-popup');
            pop.style.display = 'block';
            setTimeout(() => { pop.style.display = 'none'; }, 2000);
        }

        function loadGame() {
            const saved = localStorage.getItem('cookie_save_v1');
            if (saved) {
                const parsed = JSON.parse(saved);
                state.cookies = parsed.cookies || 0;
                state.owned = parsed.owned || {};
                console.log("Loaded save data.");
            }
            updateUI();
        }

        function buy(id) {
            const b = buildings.find(x => x.id === id);
            const cost = Math.floor(b.baseCost * Math.pow(1.15, state.owned[id]));
            if (state.cookies >= cost) {
                state.cookies -= cost;
                state.owned[id]++;
                updateUI();
                saveGame();
            }
        }

        function resetGame() {
            if(confirm("Wipe all progress?")) {
                localStorage.removeItem('cookie_save_v1');
                location.reload();
            }
        }

        window.onload = () => {
            const list = document.getElementById('shop-list');
            list.innerHTML = buildings.map(b => `
                <div class="building" id="row-${b.id}" onclick="buy('${b.id}')">
                    <div><b>${b.name}</b><br><span class="cost"></span></div>
                    <div class="count" style="font-size:1.5rem; opacity:0.3">0</div>
                </div>`).join('');
            
            document.getElementById('cookie-btn').onclick = () => {
                state.cookies++;
                updateUI();
            };

            loadGame();

            // Passive income
            setInterval(() => {
                let totalCPS = buildings.reduce((acc, b) => acc + (state.owned[b.id] * b.cps), 0);
                state.cookies += totalCPS / 10;
                updateUI();
            }, 100);

            // Auto-save every 10 seconds
            setInterval(saveGame, 10000);
        };
    </script>
</body>
</html>
