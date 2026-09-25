<script lang="ts">
    let canvas = $state<HTMLCanvasElement>();
    let ctx = $state<CanvasRenderingContext2D | null>(null);
    let offsetX = $state(0),
        offsetY = $state(0),
        isDragging = $state(false),
        startX = $state(0),
        startY = $state(0),
        zoom = $state(1),
        targetZoom = $state(1);
    let zoomFrame = 0,
        anchorX = 0,
        anchorY = 0,
        animStartTime = 0,
        animFrame = 0,
        fontsReady = false;
    const DOT_SPACING = 30,
        DOT_SIZE = 2,
        MIN_ZOOM = 0.35,
        MAX_ZOOM = 4;
    const fonts = ["GoogleFlex", "Merienda", "NotoSerif", "SourceCode"];
    const text = "nnfz";
    const imagesByLetter: Record<string, string[]> = {
        n: ["/imgs/keyboard/n.svg", "/imgs/balloon/n.png"],
        f: ["/imgs/keyboard/f.svg"],
        z: ["/imgs/keyboard/z.svg", "/imgs/balloon/z.png"],
    };
    const replaceable = [0, 1, 2, 3].filter(i => imagesByLetter[text[i]]);
    const imgSlots = new Set<number>();
    for (const i of replaceable.sort(() => Math.random() - 0.5).slice(0, Math.random() < 0.5 ? 1 : 2))
        imgSlots.add(i);
    const letterStyles = Array.from({ length: 4 }, (_, i) => {
        const imgs = imagesByLetter[text[i]];
        const imgSrc = imgSlots.has(i) && imgs ? imgs[Math.floor(Math.random() * imgs.length)] : null;
        const isKeyboard = imgSrc?.includes("keyboard");
        return {
            font: fonts[Math.floor(Math.random() * fonts.length)],
            weight: 500 + Math.floor(Math.random() * 501),
            italic: Math.random() > 0.5,
            angle: (Math.random() - 0.5) * 0.3,
            gap: Math.random() * 0.1,
            dx: 0,
            dy: 0,
            imgSrc,
            imgScale: isKeyboard ? 0.55 : 1.0,
            imgEl: null as HTMLImageElement | null,
        };
    });
    // letterPos: screen-space center of each letter after last draw, for hit testing
    const letterPos: { cx: number; cy: number; hw: number; hh: number }[] = Array.from({ length: 4 }, () => ({ cx: 0, cy: 0, hw: 0, hh: 0 }));
    let dragLetter = -1;
    function initCanvas(node: HTMLCanvasElement) {
        canvas = node;
        ctx = node.getContext("2d");
        // preload images
        for (const st of letterStyles) {
            if (st.imgSrc) {
                const img = new Image();
                img.src = st.imgSrc;
                img.onload = () => draw();
                st.imgEl = img;
            }
        }
        resizeCanvas();
        window.addEventListener("resize", resizeCanvas);
        document.fonts.ready.then(() => {
            fontsReady = true;
            animStartTime = performance.now();
            animFrame = requestAnimationFrame(animateDraw);
        });
        return {
            destroy() {
                window.removeEventListener("resize", resizeCanvas);
                cancelAnimationFrame(zoomFrame);
                cancelAnimationFrame(animFrame);
            },
        };
    }
    function animateDraw(now: number) {
        draw(now);
        const elapsed = now - animStartTime;
        const STAGGER = 120, DURATION = 500;
        if (elapsed < STAGGER * 3 + DURATION)
            animFrame = requestAnimationFrame(animateDraw);
    }
    function resizeCanvas() {
        if (!canvas) return;
        canvas.width = innerWidth;
        canvas.height = innerHeight;
        draw(performance.now());
    }
    function draw(now = performance.now()) {
        if (!ctx || !canvas) return;
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        const spacing = DOT_SPACING * zoom,
            size = DOT_SIZE * Math.max(zoom, 0.7),
            normX = ((offsetX % spacing) + spacing) % spacing,
            normY = ((offsetY % spacing) + spacing) % spacing;
        ctx.fillStyle = "#635f69";
        for (let x = -normX; x < canvas.width + spacing; x += spacing)
            for (let y = -normY; y < canvas.height + spacing; y += spacing) {
                ctx.beginPath();
                ctx.arc(x, y, size, 0, Math.PI * 2);
                ctx.fill();
            }
        drawWordmark(now);
    }
    function drawWordmark(now = performance.now()) {
        if (!ctx || !canvas || !fontsReady) return;
        ctx.save();
        ctx.translate(-offsetX, -offsetY);
        ctx.scale(zoom, zoom);
        ctx.fillStyle = "#f3ecff";
        ctx.textAlign = "left";
        ctx.textBaseline = "middle";
        const fontSize = Math.min(canvas.width * 0.24, canvas.height * 0.28);

        // measure rotated bbox extents for each letter
        const extents: { leftOffset: number; advance: number }[] = [];
        for (let i = 0; i < text.length; i++) {
            const st = letterStyles[i];
            if (st.imgEl) {
                extents.push({ leftOffset: 0, advance: fontSize });
            } else {
                ctx.font =
                    (st.italic ? "italic " : "") +
                    st.weight + " " + fontSize + "px " + st.font;
                const m = ctx.measureText(text[i]);
                const cos = Math.cos(st.angle), sin = Math.sin(st.angle);
                const leftXs = [
                    [-m.actualBoundingBoxLeft, -m.actualBoundingBoxAscent],
                    [-m.actualBoundingBoxLeft,  m.actualBoundingBoxDescent],
                ].map(([cx, cy]) => cx * cos - cy * sin);
                extents.push({
                    leftOffset: Math.min(...leftXs),
                    advance: m.width * cos,
                });
            }
        }

        // total width for centering
        let totalWidth = 0;
        for (let i = 0; i < text.length; i++) {
            totalWidth += extents[i].advance;
            if (i < text.length - 1)
                totalWidth += letterStyles[i].gap * fontSize;
        }

        const STAGGER = 120, DURATION = 500;
        let x = canvas.width / 2 - totalWidth / 2;
        for (let i = 0; i < text.length; i++) {
            const st = letterStyles[i];
            const elapsed = now - animStartTime - i * STAGGER;
            const t = Math.max(0, Math.min(1, elapsed / DURATION));
            // ease out cubic
            const ease = 1 - Math.pow(1 - t, 3);
            const scale = 1.25 - 0.25 * ease;
            const alpha = ease;
            ctx.save();
            ctx.globalAlpha = alpha;
            const cx = x - extents[i].leftOffset + st.dx;
            const cy = canvas.height / 2 + st.dy;
            ctx.translate(cx, cy);
            ctx.rotate(st.angle);
            ctx.scale(scale, scale);
            if (st.imgEl) {
                const s = fontSize * st.imgScale;
                ctx.drawImage(st.imgEl, -s * 0.5, -s * 0.5, s, s);
            } else {
                ctx.font =
                    (st.italic ? "italic " : "") +
                    st.weight + " " + fontSize + "px " + st.font;
                ctx.fillText(text[i], 0, 0);
            }
            ctx.restore();
            // record screen-space center for hit testing
            letterPos[i] = {
                cx: cx * zoom - offsetX + extents[i].advance * zoom * 0.5,
                cy: cy * zoom - offsetY,
                hw: extents[i].advance * 0.5 * zoom,
                hh: fontSize * 0.6 * zoom,
            };
            x += extents[i].advance;
            if (i < text.length - 1) x += st.gap * fontSize;
        }
        ctx.restore();
    }
    function hitLetter(ex: number, ey: number) {
        for (let i = letterPos.length - 1; i >= 0; i--) {
            const p = letterPos[i];
            if (Math.abs(ex - p.cx) < p.hw && Math.abs(ey - p.cy) < p.hh) return i;
        }
        return -1;
    }
    function handleMouseDown(e: PointerEvent) {
        if (e.button !== 0 && e.button !== 2) return;
        (e.currentTarget as HTMLCanvasElement).setPointerCapture(e.pointerId);
        startX = e.clientX;
        startY = e.clientY;
        dragLetter = hitLetter(e.clientX, e.clientY);
        isDragging = dragLetter === -1;
    }
    function handleMouseMove(e: PointerEvent) {
        const dx = e.clientX - startX;
        const dy = e.clientY - startY;
        startX = e.clientX;
        startY = e.clientY;
        if (dragLetter >= 0) {
            letterStyles[dragLetter].dx += dx / zoom;
            letterStyles[dragLetter].dy += dy / zoom;
            draw();
        } else if (isDragging) {
            offsetX -= dx;
            offsetY -= dy;
            draw();
        }
    }
    function handleMouseUp(e: PointerEvent) {
        isDragging = false;
        dragLetter = -1;
        const node = e.currentTarget as HTMLCanvasElement;
        if (node.hasPointerCapture(e.pointerId))
            node.releasePointerCapture(e.pointerId);
    }
    function animateZoom() {
        zoomFrame = 0;
        const next =
            Math.abs(targetZoom - zoom) < 0.001
                ? targetZoom
                : zoom + (targetZoom - zoom) * 0.18;
        const ratio = next / zoom;
        offsetX = -anchorX + (offsetX + anchorX) * ratio;
        offsetY = -anchorY + (offsetY + anchorY) * ratio;
        zoom = next;
        draw();
        if (next !== targetZoom) zoomFrame = requestAnimationFrame(animateZoom);
    }
    function handleWheel(e: WheelEvent) {
        e.preventDefault();
        const rect = canvas?.getBoundingClientRect();
        if (rect) {
            anchorX = e.clientX - rect.left;
            anchorY = e.clientY - rect.top;
        }
        targetZoom = Math.min(
            MAX_ZOOM,
            Math.max(MIN_ZOOM, targetZoom * Math.exp(-e.deltaY * 0.0015)),
        );
        if (!zoomFrame) zoomFrame = requestAnimationFrame(animateZoom);
    }
    function handleContextMenu(e: Event) {
        e.preventDefault();
    }
</script>

<canvas
    use:initCanvas
    onpointerdown={handleMouseDown}
    onpointermove={handleMouseMove}
    onpointerup={handleMouseUp}
    onpointercancel={handleMouseUp}
    oncontextmenu={handleContextMenu}
    onwheel={handleWheel}
></canvas>

<style>
    @font-face {
        font-family: GoogleFlex;
        src: url("/fonts/Google_Sans_Flex/GoogleSansFlex-VariableFont_GRAD,ROND,opsz,slnt,wdth,wght.ttf");
    }
    @font-face {
        font-family: Merienda;
        src: url("/fonts/Merienda/Merienda-VariableFont_wght.ttf");
    }
    @font-face {
        font-family: NotoSerif;
        src: url("/fonts/Noto_Serif/NotoSerif-VariableFont_wdth,wght.ttf");
    }
    @font-face {
        font-family: SourceCode;
        src: url("/fonts/Source_Code_Pro/SourceCodePro-VariableFont_wght.ttf");
    }
    :global(body) {
        margin: 0;
        overflow: hidden;
        background: #211f24;
        color: #f3ecff;
    }
    canvas {
        color: #635f69;
        width: 100vw;
        height: 100vh;
        touch-action: none;
    }
</style>
