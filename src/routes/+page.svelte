<script lang="ts">
 let canvas=$state<HTMLCanvasElement>(); let ctx=$state<CanvasRenderingContext2D|null>(null); let offsetX=$state(0), offsetY=$state(0), isDragging=$state(false), startX=$state(0), startY=$state(0), zoom=$state(1), targetZoom=$state(1);
 let zoomFrame=0, anchorX=0, anchorY=0;
 const DOT_SPACING=30, DOT_SIZE=2, MIN_ZOOM=.35, MAX_ZOOM=4;
 function initCanvas(node:HTMLCanvasElement){canvas=node;ctx=node.getContext('2d');resizeCanvas();window.addEventListener('resize',resizeCanvas);draw();return {destroy(){window.removeEventListener('resize',resizeCanvas);cancelAnimationFrame(zoomFrame)}}}
 function resizeCanvas(){if(!canvas)return;canvas.width=innerWidth;canvas.height=innerHeight;draw()}
 function draw(){if(!ctx||!canvas)return;ctx.clearRect(0,0,canvas.width,canvas.height);const spacing=DOT_SPACING*zoom,size=DOT_SIZE*Math.max(zoom,.7),normX=((offsetX%spacing)+spacing)%spacing,normY=((offsetY%spacing)+spacing)%spacing;ctx.fillStyle='#5C635D';for(let x=-normX;x<canvas.width+spacing;x+=spacing)for(let y=-normY;y<canvas.height+spacing;y+=spacing){ctx.beginPath();ctx.arc(x,y,size,0,Math.PI*2);ctx.fill()}}
 function handleMouseDown(e:PointerEvent){if(e.button===0||e.button===2){isDragging=true;startX=e.clientX;startY=e.clientY;(e.currentTarget as HTMLCanvasElement).setPointerCapture(e.pointerId)}}
 function handleMouseMove(e:PointerEvent){if(!isDragging)return;offsetX-=e.clientX-startX;offsetY-=e.clientY-startY;startX=e.clientX;startY=e.clientY;draw()}
 function handleMouseUp(e:PointerEvent){isDragging=false;const node=e.currentTarget as HTMLCanvasElement;if(node.hasPointerCapture(e.pointerId))node.releasePointerCapture(e.pointerId)}
 function animateZoom(){zoomFrame=0;const next=Math.abs(targetZoom-zoom)<.001?targetZoom:zoom+(targetZoom-zoom)*.18;const ratio=next/zoom;offsetX=-anchorX+(offsetX+anchorX)*ratio;offsetY=-anchorY+(offsetY+anchorY)*ratio;zoom=next;draw();if(next!==targetZoom)zoomFrame=requestAnimationFrame(animateZoom)}
 function handleWheel(e:WheelEvent){e.preventDefault();const rect=canvas?.getBoundingClientRect();if(rect){anchorX=e.clientX-rect.left;anchorY=e.clientY-rect.top}targetZoom=Math.min(MAX_ZOOM,Math.max(MIN_ZOOM,targetZoom*Math.exp(-e.deltaY*.0015)));if(!zoomFrame)zoomFrame=requestAnimationFrame(animateZoom)}
 function handleContextMenu(e:Event){e.preventDefault()}
</script>
<canvas use:initCanvas onpointerdown={handleMouseDown} onpointermove={handleMouseMove} onpointerup={handleMouseUp} onpointercancel={handleMouseUp} oncontextmenu={handleContextMenu} onwheel={handleWheel} style="display:block;cursor:{isDragging?'grabbing':'grab'};"></canvas>
<style>:global(body){margin:0;overflow:hidden;background:#1F2421}canvas{width:100vw;height:100vh;touch-action:none}</style>








