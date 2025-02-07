<!DOCTYPE html>
<html>
<head>
    <title>3D Lung Nodule Reconstruction</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/qrcode-builder/qrcode.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/dat-gui/0.7.7/dat.gui.min.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/dat-gui/0.7.7/dat.gui.css">
    <style>
        body {
            margin: 0;
            font-family: Arial, sans-serif;
            background: #f0f0f0;
        }
        #container {
            display: flex;
            height: 100vh;
        }
        #visualization {
            flex: 1;
            position: relative;
        }
        #controls {
            width: 250px;
            padding: 20px;
            background: #ffffff;
            box-shadow: 2px 0 5px rgba(0,0,0,0.1);
            overflow-y: auto;
        }
        .control-panel {
            margin-bottom: 20px;
            padding: 15px;
            background: #f8f9fa;
            border-radius: 5px;
        }
        .control-panel h3 {
            margin: 10px 0;
            color: #333;
            border-bottom: 1px solid #eee;
            padding-bottom: 5px;
        }
        button {
            padding: 8px 16px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            margin: 5px;
        }
        button:hover {
            background: #0056b3;
        }
        .qr-code {
            margin-top: 20px;
            text-align: center;
        }
        #info {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            font-size: 14px;
            background: rgba(0,0,0,0.7);
            padding: 10px;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <div id="container">
        <div id="visualization">
            <div id="info">🖱️ 拖动鼠标旋转 | 滚轮缩放 | Shift + 鼠标右键平移</div>
        </div>
        <div id="controls">
            <div class="control-panel">
                <h3>结节设置</h3>
                <div id="noduleControls"></div>
            </div>
            <div class="control-panel">
                <h3>肺段设置</h3>
                <div id="segmentControls"></div>
            </div>
            <div class="control-panel">
                <h3>血管设置</h3>
                <div id="vesselControls"></div>
            </div>
            <div class="control-panel">
                <h3>其他功能</h3>
                <button onclick="generateQRCode()">生成二维码</button>
            </div>
            <div class="qr-code" id="qrcode"></div>
        </div>
    </div>

    <script>
        // 初始化 Three.js 场景
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('visualization').appendChild(renderer.domElement);

        // 示例数据：肺结节和肺段信息
        const lungData = {
            nodules: [
                { size: 3, position: [0, 0, 0] },      // 低危结节（绿色）
                { size: 6, position: [2, 0, 0] },      // 中危结节（黄色）
                { size: 8, position: [-2, 0, 0] },     // 高危结节（紫色）
            ],
            segments: [
                { color: 0x00ff00, opacity: 0.3 },      // 肺段1
                { color: 0xffff00, opacity: 0.3 },      // 肺段2
                { color: 0xff00ff, opacity: 0.3 },      // 肺段3
            ],
            vessels: {
                artery: { color: 0x0000ff, visible: true },           // 肺动脉（蓝色）
                vein: { color: 0xff0000, visible: true }              // 肺静脉（红色）
            }
        };

        // 创建 3D 对象
        function createNoduleGeometry(size, color) {
            const geometry = new THREE.SphereGeometry(size);
            const material = new THREE.MeshBasicMaterial({ color: color });
            return new THREE.Mesh(geometry, material);
        }

        // 根据大小着色
        function getColorBySize(size) {
            if (size < 4) return 0x00ff00; // 绿色
            if (size >= 5 && size < 8) return 0xffff00; // 黄色
            return 0x800080; // 紫色
        }

        // 创建肺结节
        const noduleMeshes = lungData.nodules.map(nodule => {
            const color = getColorBySize(nodule.size);
            const mesh = createNoduleGeometry(nodule.size, color);
            scene.add(mesh);
            return mesh;
        });

        // 创建肺段（示例）
        const segmentMeshes = lungData.segments.map((segment, index) => {
            const geometry = new THREE.BoxGeometry(1, 1, 1);
            const material = new THREE.MeshBasicMaterial({
                color: segment.color,
                transparent: true,
                opacity: segment.opacity
            });
            const mesh = new THREE.Mesh(geometry, material);
            scene.add(mesh);
            return mesh;
        });

        // 创建肺动脉和肺静脉（示例）
        const arteryGeometry = new THREE.TorusGeometry(1, 0.2, 16, 100);
        const arteryMaterial = new THREE.MeshBasicMaterial({ color: lungData.vessels.artery.color });
        const artery = new THREE.Mesh(arteryGeometry, arteryMaterial);
        scene.add(artery);

        const veinGeometry = new THREE.TorusGeometry(1, 0.2, 16, 100);
        const veinMaterial = new THREE.MeshBasicMaterial({ color: lungData.vessels.vein.color });
        const vein = new THREE.Mesh(veinGeometry, veinMaterial);
        scene.add(vein);

        // 创建控制面板
        const gui = new dat.GUI({
            container: document.getElementById('noduleControls')
        });

        // 结节控制面板
        const noduleParams = {
            showNodules: true,
            noduleOpacity: 1,
            lowRiskColor: '#00ff00',
            mediumRiskColor: '#ffff00',
            highRiskColor: '#800080'
        };

        gui.add(noduleParams, 'showNodules', true, false, '显示结节')
            .onChange(() => {
                noduleMeshes.forEach(mesh => mesh.visible = noduleParams.showNodules);
            });

        gui.add(noduleParams, 'noduleOpacity', 0, 1, 0.1)
            .onChange(() => {
                noduleMeshes.forEach(mesh => {
                    mesh.material.opacity = noduleParams.noduleOpacity;
                    mesh.material.transparent = noduleParams.noduleOpacity < 1;
                });
            });

        gui.addColor(noduleParams, 'lowRiskColor')
            .onChange(() => {
                noduleMeshes.forEach((mesh, index) => {
                    if (lungData.nodules[index].size < 4) {
                        mesh.material.color.set(noduleParams.lowRiskColor);
                    }
                });
            });

        gui.addColor(noduleParams, 'mediumRiskColor')
            .onChange(() => {
                noduleMeshes.forEach((mesh, index) => {
                    if (lungData.nodules[index].size >= 5 && lungData.nodules[index].size < 8) {
                        mesh.material.color.set(noduleParams.mediumRiskColor);
                    }
                });
            });

        gui.addColor(noduleParams, 'highRiskColor')
            .onChange(() => {
                noduleMeshes.forEach((mesh, index) => {
                    if (lungData.nodules[index].size >= 8) {
                        mesh.material.color.set(noduleParams.highRiskColor);
                    }
                });
            });

        // 肺段控制面板
        const segmentGui = new dat.GUI({
            container: document.getElementById('segmentControls')
        });

        const segmentParams = {
            showSegments: true,
            segmentOpacity: 0.3
        };

        segmentGui.add(segmentParams, 'showSegments', true, false, '显示肺段')
            .onChange(() => {
                segmentMeshes.forEach(mesh => mesh.visible = segmentParams.showSegments);
            });

        segmentGui.add(segmentParams, 'segmentOpacity', 0, 1, 0.1)
            .onChange(() => {
                segmentMeshes.forEach(mesh => {
                    mesh.material.opacity = segmentParams.segmentOpacity;
                    mesh.material.transparent = segmentParams.segmentOpacity < 1;
                });
            });

        // 血管控制面板
        const vesselGui = new dat.GUI({
            container: document.getElementById('vesselControls')
        });

        const vesselParams = {
            showArtery: lungData.vessels.artery.visible,
            showVein: lungData.vessels.vein.visible,
            arteryColor: lungData.vessels.artery.color,
            veinColor: lungData.vessels.vein.color
        };

        vesselGui.add(vesselParams, 'showArtery', true, false, '显示肺动脉')
            .onChange(() => artery.visible = vesselParams.showArtery);

        vesselGui.add(vesselParams, 'showVein', true, false, '显示肺静脉')
            .onChange(() => vein.visible = vesselParams.showVein);

        vesselGui.addColor(vesselParams, 'arteryColor')
            .onChange(() => artery.material.color.set(vesselParams.arteryColor));

        vesselGui.addColor(vesselParams, 'veinColor')
            .onChange(() => vein.material.color.set(vesselParams.veinColor));

        // 相机位置
        camera.position.z = 10;

        // 动画循环
        function animate() {
            requestAnimationFrame(animate);
            renderer.render(scene, camera);
        }

        // 生成二维码
        function generateQRCode() {
            const text = "https://example.com/3d-model"; // 替换为实际的 3D 模型 URL
            const qr = qrcode.create(text);
            const qrImage = document.createElement('img');
            qrImage.src = QRCode.toCanvas(qr, { margin: 4 });
            document.getElementById('qrcode').appendChild(qrImage);
        }

        // 初始化控制面板
        animate();

        // 添加窗口resize事件监听
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
