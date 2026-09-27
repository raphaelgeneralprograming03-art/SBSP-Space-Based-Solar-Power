
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simulação: Energia Solar Global & Orbital</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            user-select: none;
        }

        body {
            background-color: #050811;
            color: #e0f7fc;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            display: flex;
            height: 100vh;
            width: 100vw;
        }

        #canvas-container {
            flex: 1;
            position: relative;
            height: 100%;
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        /* Painel de Controle e Métricas */
        #ui-panel {
            width: 340px;
            background: rgba(10, 16, 30, 0.85);
            backdrop-filter: blur(12px);
            border-left: 1px solid rgba(0, 229, 255, 0.2);
            padding: 20px;
            display: flex;
            flex-direction: column;
            gap: 18px;
            box-shadow: -5px 0 25px rgba(0, 0, 0, 0.5);
            z-index: 10;
        }

        h1 {
            font-size: 1.1rem;
            letter-spacing: 1.5px;
            text-transform: uppercase;
            color: #00e5ff;
            border-bottom: 1px solid rgba(0, 229, 255, 0.3);
            padding-bottom: 8px;
        }

        .metric-card {
            background: rgba(255, 255, 255, 0.03);
            border: 1px solid rgba(255, 255, 255, 0.08);
            border-radius: 8px;
            padding: 12px;
        }

        .metric-title {
            font-size: 0.75rem;
            color: #8ab4f8;
            text-transform: uppercase;
            margin-bottom: 4px;
        }

        .metric-value {
            font-size: 1.4rem;
            font-weight: bold;
            font-family: monospace;
            color: #ffffff;
        }

        .metric-sub {
            font-size: 0.7rem;
            color: #00e676;
            margin-top: 2px;
        }

        .control-group {
            display: flex;
            flex-direction: column;
            gap: 6px;
        }

        label {
            font-size: 0.8rem;
            color: #cfd8dc;
            display: flex;
            justify-content: space-between;
        }

        input[type="range"] {
            accent-color: #00e5ff;
            cursor: pointer;
        }

        .legend {
            display: flex;
            flex-direction: column;
            gap: 8px;
            font-size: 0.75rem;
        }

        .legend-item {
            display: flex;
            align-items: center;
            gap: 8px;
        }

        .dot {
            width: 10px;
            height: 10px;
            border-radius: 50%;
        }

        .dot-sun { background: #ffea00; box-shadow: 0 0 8px #ffea00; }
        .dot-sat { background: #00e5ff; box-shadow: 0 0 8px #00e5ff; }
        .dot-ground { background: #ff9100; box-shadow: 0 0 8px #ff9100; }
        .dot-beam { background: #00e676; box-shadow: 0 0 8px #00e676; }
    </style>
</head>
<body>

    <div id="canvas-container">
        <canvas id="simCanvas"></canvas>
    </div>

    <div id="ui-panel">
        <h1>Energia Solar Global</h1>

        <!-- Métricas em Tempo Real -->
        <div class="metric-card">
            <div class="metric-title">Geração Espacial (SBSP)</div>
            <div class="metric-value" id="orbitalPower">0 GW</div>
            <div class="metric-sub">Captação 24/7 sem atmosfera</div>
        </div>

        <div class="metric-card">
            <div class="metric-title">Geração Terrestre</div>
            <div class="metric-value" id="groundPower">0 GW</div>
            <div class="metric-sub" id="groundStatus">Afetada por noite/nuvens</div>
        </div>

        <div class="metric-card">
            <div class="metric-title">Matriz Energética Total</div>
            <div class="metric-value" id="totalPower" style="color: #00e5ff;">0 GW</div>
        </div>

        <!-- Controles Interativos -->
        <div class="control-group">
            <label>Nº de Satélites Orbitais: <span id="satCountVal">8</span></label>
            <input type="range" id="satSlider" min="2" max="24" value="8">
        </div>

        <div class="control-group">
            <label>Altitude Orbital (km): <span id="altitudeVal">8000</span></label>
            <input type="range" id="altitudeSlider" min="4000" max="14000" step="500" value="8000">
        </div>

        <div class="control-group">
            <label>Velocidade de Rotação Terrestre</label>
            <input type="range" id="speedSlider" min="1" max="10" value="3">
        </div>

        <!-- Legenda -->
        <div class="legend">
            <div class="legend-item"><div class="dot dot-sun"></div> Radiação Solar Direta</div>
            <div class="legend-item"><div class="dot dot-sat"></div> Satélite Coletor Orbital</div>
            <div class="legend-item"><div class="dot dot-beam"></div> Feixe Micro-ondas/Laser</div>
            <div class="legend-item"><div class="dot dot-ground"></div> Receptora de Superfície</div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('simCanvas');
        const ctx = canvas.getContext('2d');

        // Configuração Responsiva
        function resizeCanvas() {
            canvas.width = canvas.parentElement.clientWidth;
            canvas.height = canvas.parentElement.clientHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        // Parâmetros Globais da Simulação
        const state = {
            earthRadius: 110,
            atmoThickness: 28,
            sunAngle: Math.PI, // Luz solar vinda da esquerda
            earthRotation: 0,
            rotationSpeed: 0.003,
            satellitesCount: 8,
            orbitRadius: 210,
            groundStationsCount: 12,
            particles: []
        };

        // Elementos DOM
        const satSlider = document.getElementById('satSlider');
        const altitudeSlider = document.getElementById('altitudeSlider');
        const speedSlider = document.getElementById('speedSlider');
        
        satSlider.addEventListener('input', (e) => {
            state.satellitesCount = parseInt(e.target.value);
            document.getElementById('satCountVal').innerText = state.satellitesCount;
        });

        altitudeSlider.addEventListener('input', (e) => {
            const val = parseInt(e.target.value);
            state.orbitRadius = state.earthRadius + (val / 100);
            document.getElementById('altitudeVal').innerText = val;
        });

        speedSlider.addEventListener('input', (e) => {
            state.rotationSpeed = parseInt(e.target.value) * 0.001;
        });

        // Estrutura das Estações Terrestres
        class GroundStation {
            constructor(angleIndex, total) {
                this.angleOffset = (Math.PI * 2 / total) * angleIndex;
                this.capacityGW = 15; // Capacidade por estação
            }

            getPos(cx, cy) {
                const currentAngle = this.angleOffset + state.earthRotation;
                return {
                    x: cx + Math.cos(currentAngle) * state.earthRadius,
                    y: cy + Math.sin(currentAngle) * state.earthRadius,
                    angle: currentAngle
                };
            }
        }

        // Estações terrestres fixas no Equador do planeta
        let groundStations = [];
        function initGroundStations() {
            groundStations = [];
            for (let i = 0; i < state.groundStationsCount; i++) {
                groundStations.push(new GroundStation(i, state.groundStationsCount));
            }
        }
        initGroundStations();

        // Partículas de Energia no Feixe
        function createBeamParticle(startX, startY, endX, endY) {
            state.particles.push({
                x: startX,
                y: startY,
                startX, startY, endX, endY,
                progress: 0,
                speed: 0.03 + Math.random() * 0.02
            });
        }

        // Loop Principal de Renderização
        function animate() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            const cx = canvas.width / 2;
            const cy = canvas.height / 2;

            // Atualiza rotação da Terra
            state.earthRotation += state.rotationSpeed;

            // 1. Desenhar Fundo Estelar Simples
            drawStars();

            // 2. Desenhar Raios Solares Vindo do Sol (Esquerda)
            drawSunlight(cx, cy);

            // 3. Desenhar Atmosfera Terrestre
            drawAtmosphere(cx, cy);

            // 4. Desenhar Terra (Superfície + Sombra Dia/Noite)
            drawEarth(cx, cy);

            // 5. Atualizar e Desenhar Estações Terrestres
            const groundMetrics = updateAndDrawGroundStations(cx, cy);

            // 6. Atualizar e Desenhar Constelação Orbital
            const orbitalMetrics = updateAndDrawSatellites(cx, cy, groundMetrics.activeStations);

            // 7. Atualizar Partículas de Feixes de Energia
            updateParticles();

            // 8. Atualizar Dashboard
            document.getElementById('orbitalPower').innerText = `${orbitalMetrics.totalPower.toFixed(1)} GW`;
            document.getElementById('groundPower').innerText = `${groundMetrics.totalPower.toFixed(1)} GW`;
            document.getElementById('totalPower').innerText = `${(orbitalMetrics.totalPower + groundMetrics.totalPower).toFixed(1)} GW`;

            requestAnimationFrame(animate);
        }

        // Fundo com Estrelas
        function drawStars() {
            ctx.fillStyle = "rgba(255, 255, 255, 0.4)";
            for (let i = 0; i < 50; i++) {
                const x = (Math.sin(i * 99) * 0.5 + 0.5) * canvas.width;
                const y = (Math.cos(i * 33) * 0.5 + 0.5) * canvas.height;
                ctx.fillRect(x, y, 1.5, 1.5);
            }
        }

        // Luz Solar Direta
        function drawSunlight(cx, cy) {
            ctx.save();
            const grad = ctx.createLinearGradient(0, cy, cx - state.earthRadius, cy);
            grad.addColorStop(0, "rgba(255, 234, 0, 0.15)");
            grad.addColorStop(1, "rgba(255, 234, 0, 0.0)");

            ctx.fillStyle = grad;
            ctx.fillRect(0, cy - state.orbitRadius - 50, cx - state.earthRadius, (state.orbitRadius + 50) * 2);
            ctx.restore();
        }

        // Atmosfera com Efeito Glow
        function drawAtmosphere(cx, cy) {
            ctx.save();
            ctx.beginPath();
            ctx.arc(cx, cy, state.earthRadius + state.atmoThickness, 0, Math.PI * 2);
            const atmoGrad = ctx.createRadialGradient(
                cx, cy, state.earthRadius,
                cx, cy, state.earthRadius + state.atmoThickness
            );
            atmoGrad.addColorStop(0, "rgba(0, 229, 255, 0.35)");
            atmoGrad.addColorStop(1, "rgba(0, 229, 255, 0.0)");
            ctx.fillStyle = atmoGrad;
            ctx.fill();
            ctx.restore();
        }

        // Planeta Terra e Sombra
        function drawEarth(cx, cy) {
            ctx.save();
            
            // Oceano Base
            ctx.beginPath();
            ctx.arc(cx, cy, state.earthRadius, 0, Math.PI * 2);
            ctx.fillStyle = "#0d47a1";
            ctx.fill();

            // Continentes Ilustrativos Rotativos
            ctx.clip();
            ctx.fillStyle = "#2e7d32";
            for (let i = 0; i < 6; i++) {
                const angle = state.earthRotation + (i * Math.PI / 3);
                const continentX = cx + Math.cos(angle) * (state.earthRadius * 0.6);
                const continentY = cy + Math.sin(angle) * (state.earthRadius * 0.6);
                
                ctx.beginPath();
                ctx.arc(continentX, continentY, 25 + (i * 5), 0, Math.PI * 2);
                ctx.fill();
            }

            // Sombra da Noite (Hemisfério Oposto ao Sol)
            ctx.beginPath();
            ctx.rect(cx, cy - state.earthRadius - 5, state.earthRadius + 10, (state.earthRadius + 5) * 2);
            const shadowGrad = ctx.createLinearGradient(cx, cy, cx + state.earthRadius, cy);
            shadowGrad.addColorStop(0, "rgba(5, 8, 17, 0.4)");
            shadowGrad.addColorStop(0.4, "rgba(5, 8, 17, 0.88)");
            shadowGrad.addColorStop(1, "rgba(5, 8, 17, 0.98)");
            ctx.fillStyle = shadowGrad;
            ctx.fill();

            ctx.restore();

            // Borda do Planeta
            ctx.beginPath();
            ctx.arc(cx, cy, state.earthRadius, 0, Math.PI * 2);
            ctx.strokeStyle = "rgba(255, 255, 255, 0.1)";
            ctx.stroke();
        }

        // Estações na Superfície
        function updateAndDrawGroundStations(cx, cy) {
            let totalPower = 0;
            const activeStations = [];

            groundStations.forEach(st => {
                const pos = st.getPos(cx, cy);
                
                // Determina se a estação está no lado do Dia (x < cx)
                const isDaytime = pos.x < cx;
                // Atenuação atmosférica simples baseada no ângulo com o Sol
                const sunExposure = isDaytime ? Math.max(0, (cx - pos.x) / state.earthRadius) : 0;
                
                const currentPower = sunExposure * st.capacityGW;
                totalPower += currentPower;

                activeStations.push({ ...pos, isDaytime, instance: st });

                // Renderizar Coletor Terrestre
                ctx.save();
                ctx.beginPath();
                ctx.arc(pos.x, pos.y, 4, 0, Math.PI * 2);
                ctx.fillStyle = isDaytime ? "#ff9100" : "#546e7a";
                ctx.shadowColor = isDaytime ? "#ff9100" : "transparent";
                ctx.shadowBlur = 8;
                ctx.fill();
                ctx.restore();
            });

            return { totalPower, activeStations };
        }

        // Satélites Orbitais e Feixes Seletores
        function updateAndDrawSatellites(cx, cy, groundStationsPos) {
            let totalPower = 0;

            // Linha Guia da Órbita
            ctx.beginPath();
            ctx.arc(cx, cy, state.orbitRadius, 0, Math.PI * 2);
            ctx.strokeStyle = "rgba(0, 229, 255, 0.12)";
            ctx.setLineDash([4, 6]);
            ctx.stroke();
            ctx.setLineDash([]);

            const step = (Math.PI * 2) / state.satellitesCount;
            const satOrbitSpeed = state.earthRotation * 0.5; // Órbita em velocidade própria

            for (let i = 0; i < state.satellitesCount; i++) {
                const satAngle = (i * step) + satOrbitSpeed;
                const satX = cx + Math.cos(satAngle) * state.orbitRadius;
                const satY = cy + Math.sin(satAngle) * state.orbitRadius;

                // Verificação de Eclipse (Sombra da Terra)
                // O eclipse ocorre quando o satélite está atrás da Terra em relação ao Sol
                const inEclipse = (satX > cx) && (Math.abs(satY - cy) < state.earthRadius);

                const powerGenerated = inEclipse ? 0 : 25; // 25 GW contínuos fora do eclipse
                totalPower += powerGenerated;

                // Renderiza Satélite
                ctx.save();
                ctx.translate(satX, satY);

                // Painéis Solares (Sempre voltados para o Sol na esquerda)
                ctx.fillStyle = "#00e5ff";
                ctx.fillRect(-6, -8, 3, 16);
                ctx.fillRect(3, -8, 3, 16);
                
                // Corpo Central
                ctx.fillStyle = "#ffffff";
                ctx.beginPath();
                ctx.arc(0, 0, 3, 0, Math.PI * 2);
                ctx.fill();

                if (!inEclipse) {
                    ctx.shadowColor = "#00e5ff";
                    ctx.shadowBlur = 10;
                }
                ctx.restore();

                // Transmissão de Energia via Feixe Seletor (SBSP)
                if (!inEclipse) {
                    // Encontrar a Estação Terrestre mais próxima visível
                    let closestStation = null;
                    let minDist = Infinity;

                    groundStationsPos.forEach(stPos => {
                        // Linha de Visada: distância escalar simples entre satélite e estação
                        const dist = Math.hypot(stPos.x - satX, stPos.y - satY);
                        
                        // Checar se a Terra não bloqueia o feixe
                        const midX = (satX + stPos.x) / 2;
                        const midY = (satY + stPos.y) / 2;
                        const distToCenter = Math.hypot(midX - cx, midY - cy);

                        if (distToCenter > state.earthRadius - 5 && dist < minDist) {
                            minDist = dist;
                            closestStation = stPos;
                        }
                    });

                    // Desenhar Feixe Mico-ondas/Laser para a estação receptora
                    if (closestStation) {
                        ctx.save();
                        ctx.beginPath();
                        ctx.moveTo(satX, satY);
                        ctx.lineTo(closestStation.x, closestStation.y);
                        
                        // Estilização do Feixe Seletor
                        const beamGrad = ctx.createLinearGradient(satX, satY, closestStation.x, closestStation.y);
                        beamGrad.addColorStop(0, "rgba(0, 230, 118, 0.8)");
                        beamGrad.addColorStop(1, "rgba(0, 229, 255, 0.2)");

                        ctx.strokeStyle = beamGrad;
                        ctx.lineWidth = 1.5;
                        ctx.shadowColor = "#00e676";
                        ctx.shadowBlur = 8;
                        ctx.stroke();
                        ctx.restore();

                        // Gerar partículas de fluxo energético
                        if (Math.random() < 0.3) {
                            createBeamParticle(satX, satY, closestStation.x, closestStation.y);
                        }
                    }
                }
            }

            return { totalPower };
        }

        // Animação das Partículas do Feixe Seletor
        function updateParticles() {
            for (let i = state.particles.length - 1; i >= 0; i--) {
                const p = state.particles[i];
                p.progress += p.speed;

                if (p.progress >= 1) {
                    state.particles.splice(i, 1);
                    continue;
                }

                p.x = p.startX + (p.endX - p.startX) * p.progress;
                p.y = p.startY + (p.endY - p.startY) * p.progress;

                ctx.save();
                ctx.beginPath();
                ctx.arc(p.x, p.y, 1.5, 0, Math.PI * 2);
                ctx.fillStyle = "#ffffff";
                ctx.shadowColor = "#00e676";
                ctx.shadowBlur = 5;
                ctx.fill();
                ctx.restore();
            }
        }

        // Iniciar Simulação
        animate();
    </script>
</body>
</html>
