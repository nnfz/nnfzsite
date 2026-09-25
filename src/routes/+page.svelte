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
        anchorY = 0;
    const DOT_SPACING = 30,
        DOT_SIZE = 2,
        MIN_ZOOM = 0.35,
        MAX_ZOOM = 4;
    const fonts = ["GoogleFlex", "Merienda", "NotoSerif", "SourceCode"];
    const letterStyles = Array.from({ length: 4 }, () => ({
        font: fonts[Math.floor(Math.random() * fonts.length)],
        weight: 500 + Math.floor(Math.random() * 501),
        italic: Math.random() > 0.5,
        angle: (Math.random() - 0.5) * 0.3,
        gap: Math.random() * 0.1,
    }));
    function initCanvas(node: HTMLCanvasElement) {
        canvas = node;
        ctx = node.getContext("2d");
        resizeCanvas();
        window.addEventListener("resize", resizeCanvas);
        document.fonts.ready.then(draw);
        return {
            destroy() {
                window.removeEventListener("resize", resizeCanvas);
                cancelAnimationFrame(zoomFrame);
            },
        };
    }
    function resizeCanvas() {
        if (!canvas) return;
        canvas.width = innerWidth;
        canvas.height = innerHeight;
        draw();
    }
    function draw() {
        if (!ctx || !canvas) return;
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        const spacing = DOT_SPACING * zoom,
            size = DOT_SIZE * Math.max(zoom, 0.7),
            normX = ((offsetX % spacing) + spacing) % spacing,
            normY = ((offsetY % spacing) + spacing) % spacing;
        ctx.fillStyle = "#5C635D";
        for (let x = -normX; x < canvas.width + spacing; x += spacing)
            for (let y = -normY; y < canvas.height + spacing; y += spacing) {
                ctx.beginPath();
                ctx.arc(x, y, size, 0, Math.PI * 2);
                ctx.fill();
            }
        drawWordmark();
    }
    function drawWordmark() {
        if (!ctx || !canvas) return;
        ctx.save();
        ctx.translate(-offsetX, -offsetY);
        ctx.scale(zoom, zoom);
        ctx.fillStyle = "#f3ecff";
        ctx.textAlign = "left";
        ctx.textBaseline = "middle";
        const text = "nnfz",
            fontSize = Math.min(canvas.width * 0.24, canvas.height * 0.28);

        // measure rotated bbox extents for each letter
        const extents: { leftOffset: number; advance: number }[] = [];
        for (let i = 0; i < text.length; i++) {
            const st = letterStyles[i];
            ctx.font =
                (st.italic ? "italic " : "") +
                st.weight + " " + fontSize + "px " + st.font;
            const m = ctx.measureText(text[i]);
            const cos = Math.cos(st.angle), sin = Math.sin(st.angle);
            // left visual offset: rotate only the left corners
            const leftXs = [
                [-m.actualBoundingBoxLeft, -m.actualBoundingBoxAscent],
                [-m.actualBoundingBoxLeft,  m.actualBoundingBoxDescent],
            ].map(([cx, cy]) => cx * cos - cy * sin);
            extents.push({
                leftOffset: Math.min(...leftXs),
                // advance along X using typographic width, not visual bbox
                advance: m.width * cos,
            });
        }

        // total width for centering
        let totalWidth = 0;
        for (let i = 0; i < text.length; i++) {
            totalWidth += extents[i].advance;
            if (i < text.length - 1)
                totalWidth += letterStyles[i].gap * fontSize;
        }

        let x = canvas.width / 2 - totalWidth / 2;
        for (let i = 0; i < text.length; i++) {
            const st = letterStyles[i];
            ctx.save();
            ctx.translate(x - extents[i].leftOffset, canvas.height / 2);
            ctx.rotate(st.angle);
            ctx.font =
                (st.italic ? "italic " : "") +
                st.weight + " " + fontSize + "px " + st.font;
            ctx.fillText(text[i], 0, 0);
            ctx.restore();
            x += extents[i].advance;
            if (i < text.length - 1) x += st.gap * fontSize;
        }
        ctx.restore();
    }
    function handleMouseDown(e: PointerEvent) {
        if (e.button === 0 || e.button === 2) {
            isDragging = true;
            startX = e.clientX;
            startY = e.clientY;
            (e.currentTarget as HTMLCanvasElement).setPointerCapture(
                e.pointerId,
            );
        }
    }
    function handleMouseMove(e: PointerEvent) {
        if (!isDragging) return;
        offsetX -= e.clientX - startX;
        offsetY -= e.clientY - startY;
        startX = e.clientX;
        startY = e.clientY;
        draw();
    }
    function handleMouseUp(e: PointerEvent) {
        isDragging = false;
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
        color: #f3ecff;
        width: 100vw;
        height: 100vh;
        touch-action: none;
    }
</style>
