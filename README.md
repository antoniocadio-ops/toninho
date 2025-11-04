<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Financial Spoofing Simulation</title>
<script type="importmap">
{
  "imports": {
    "chart.js": "https://cdn.jsdelivr.net/npm/chart.js",
    "d3": "https://cdn.jsdelivr.net/npm/d3@7/+esm"
  }
}
</script>
<style>
  body { font-family: Arial, sans-serif; background: #101820; color: #eee; display: flex; flex-direction: column; align-items: center; }
  #container { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-top: 20px; }
  .panel { background: #18222f; padding: 16px; border-radius: 10px; box-shadow: 0 0 10px #000; }
  #chartsPanel { grid-column: span 2; display: flex; justify-content: space-around; }
  canvas { background: #0f1622; border-radius: 8px; }
  #personMonitor { position: relative; text-align: center; }
  #monitor { background: #000; width: 180px; height: 120px; margin: auto; border-radius: 8px; border: 3px solid #555; overflow: hidden; }
  #monitor canvas { width: 100%; height: 100%; }
  svg { width: 100px; }
  #fraudDetection { grid-column: span 2; height: 300px; position: relative; }
  .tooltip { position: absolute; background: rgba(0,0,0,0.8); color: #fff; padding: 6px; border-radius: 6px; pointer-events: none; font-size: 0.8em; }
  #logPanel { grid-column: span 2; max-height: 200px; overflow-y: auto; background: #0f1622; border-radius: 8px; padding: 10px; font-size: 0.9em; }
  .logEntry { border-bottom: 1px solid #333; padding: 4px 0; display: flex; justify-content: space-between; align-items: center; }
  .severity { padding: 2px 6px; border-radius: 4px; font-size: 0.8em; font-weight: bold; }
  .low { background: #2a4; color: #fff; }
  .medium { background: #eaa800; color: #fff; }
  .critical { background: #c22; color: #fff; }
  @keyframes glow { from { filter: drop-shadow(0 0 0px red); } to { filter: drop-shadow(0 0 20px red); } }
  @keyframes shake { 0%,100% { transform: translate(0,0); } 25% { transform: translate(-3px,0); } 50% { transform: translate(3px,0); } 75% { transform: translate(-3px,0); } }
  .glow { animation: glow 0.8s alternate infinite; }
  .shake { animation: shake 0.4s linear infinite; }
</style>
</head>
<body>
<h1>Financial Spoofing Simulation</h1>
<div id="container">
  <div class="panel" id="orderBookPanel"><h2>Order Book</h2><canvas id="orderBookCanvas" width="400" height="200"></canvas></div>
  <div class="panel" id="controlsPanel">
    <h2>Controls</h2>
    <button id="buyBtn">Buy</button>
    <button id="sellBtn">Sell</button>
    <button id="spoofBtn">Spoof</button>
  </div>
  <div class="panel" id="chartsPanel">
    <canvas id="buyChart" width="300" height="150"></canvas>
    <canvas id="sellChart" width="300" height="150"></canvas>
  </div>
  <div class="panel" id="personMonitor">
    <svg viewBox="0 0 100 100"><circle cx="50" cy="30" r="20" fill="#6cf" /><rect x="35" y="50" width="30" height="40" rx="8" fill="#6cf" /></svg>
    <div id="monitor"><canvas id="liveMonitorChart"></canvas></div>
  </div>
  <div class="panel" id="fraudDetection">
    <h2>Fraud Detection Network</h2>
    <svg id="fraudNetwork"></svg>
    <div class="tooltip" id="tooltip" style="opacity:0"></div>
  </div>
  <div class="panel" id="logPanel"><h2>Fraud Event Log</h2><div id="logEntries"></div></div>
</div>
<script type="module">
import Chart from 'chart.js';
import * as d3 from 'd3';

const buyCtx = document.getElementById('buyChart');
const sellCtx = document.getElementById('sellChart');
const monitorCtx = document.getElementById('liveMonitorChart');

const buyChart = new Chart(buyCtx, { type: 'line', data: { labels: [], datasets: [{ label: 'Buys', data: [], borderColor: 'green' }] }, options: { animation: false, scales: { x: { display:false }, y: { display:true }}}});
const sellChart = new Chart(sellCtx, { type: 'line', data: { labels: [], datasets: [{ label: 'Sells', data: [], borderColor: 'red' }] }, options: { animation: false, scales: { x: { display:false }, y: { display:true }}}});
const monitorChart = new Chart(monitorCtx, { type: 'line', data: { labels: [], datasets: [{ label:'Price', data: [], borderColor:'#6cf' }]}, options: { animation:false, scales:{ x:{display:false}, y:{display:false}}}});

function addData(chart,val){chart.data.labels.push('');chart.data.datasets[0].data.push(val);if(chart.data.labels.length>20){chart.data.labels.shift();chart.data.datasets[0].data.shift();}chart.update();}

document.getElementById('buyBtn').onclick=()=>{const val=Math.random()*10+50;addData(buyChart,val);addData(monitorChart,val);triggerFraudNetwork('buy');logFraudEvent('Normal buy transaction recorded.','low');};
document.getElementById('sellBtn').onclick=()=>{const val=Math.random()*10+40;addData(sellChart,val);addData(monitorChart,val);triggerFraudNetwork('sell');logFraudEvent('Sell activity noted, monitoring for irregularities.','medium');};
document.getElementById('spoofBtn').onclick=()=>{const val=Math.random()*30+80;addData(buyChart,val);addData(sellChart,val);addData(monitorChart,val);triggerFraudNetwork('spoof');logFraudEvent('Spoofing pattern detected, regulator alerted.','critical');};

const svg = d3.select('#fraudNetwork');
const width = document.getElementById('fraudNetwork').clientWidth;
const height = document.getElementById('fraudNetwork').clientHeight;
svg.attr('width', width).attr('height', height);
const tooltip = d3.select('#tooltip');

let nodes = [ {id:'Investor'}, {id:'Bot'}, {id:'Exchange'}, {id:'Regulator'} ];
let links = [ {source:'Investor',target:'Exchange'}, {source:'Bot',target:'Exchange'}, {source:'Exchange',target:'Regulator'} ];
const colorScale = d3.scaleOrdinal().domain(['buy','sell','spoof']).range(['#0f0','#f33','#ff0']);

const simulation = d3.forceSimulation(nodes).force('link', d3.forceLink(links).id(d=>d.id).distance(100)).force('charge', d3.forceManyBody().strength(-300)).force('center', d3.forceCenter(width/2, height/2));

const link = svg.append('g').selectAll('line').data(links).enter().append('line').attr('stroke','#888');
const node = svg.append('g').selectAll('circle').data(nodes).enter().append('circle').attr('r',20).attr('fill','#444').call(drag(simulation));
const label = svg.append('g').selectAll('text').data(nodes).enter().append('text').text(d=>d.id).attr('dy',5).attr('x',-15).attr('fill','#fff');

node.on('mouseover',(event,d)=>{tooltip.style('opacity',1).html(`<b>${d.id}</b>`).style('left',(event.pageX+10)+'px').style('top',(event.pageY-10)+'px');})
    .on('mouseout',()=>tooltip.style('opacity',0))
    .on('click',(event,d)=>{alert(`Inspecting ${d.id} for suspicious activity...`);logFraudEvent(`Inspection started on ${d.id}.`,'medium');});

simulation.on('tick',()=>{
  link.attr('x1',d=>d.source.x).attr('y1',d=>d.source.y).attr('x2',d=>d.target.x).attr('y2',d=>d.target.y);
  node.attr('cx',d=>d.x).attr('cy',d=>d.y);
  label.attr('x',d=>d.x-15).attr('y',d=>d.y+5);
});

function drag(simulation){
  function dragstarted(event){if(!event.active)simulation.alphaTarget(0.3).restart();event.subject.fx=event.subject.x;event.subject.fy=event.subject.y;}
  function dragged(event){event.subject.fx=event.x;event.subject.fy=event.y;}
  function dragended(event){if(!event.active)simulation.alphaTarget(0);event.subject.fx=null;event.subject.fy=null;}
  return d3.drag().on('start',dragstarted).on('drag',dragged).on('end',dragended);
}

function triggerFraudNetwork(type){
  const c=colorScale(type);
  node.transition().duration(300).attr('fill',(d,i)=>i%2===0?c:'#555');
  svg.append('circle').attr('cx',width/2).attr('cy',height/2).attr('r',0).attr('fill',c).attr('opacity',0.3).transition().duration(600).attr('r',150).attr('opacity',0).remove();
}

function visualAlert(level){
  const allNodes=document.querySelectorAll('#fraudNetwork circle');
  allNodes.forEach(n=>{n.classList.remove('glow','shake');});
  if(level==='medium'){
    allNodes.forEach(n=>n.classList.add('glow'));
    setTimeout(()=>allNodes.forEach(n=>n.classList.remove('glow')),2000);
  } else if(level==='critical'){
    allNodes.forEach(n=>n.classList.add('shake','glow'));
    setTimeout(()=>allNodes.forEach(n=>n.classList.remove('shake','glow')),2000);
  }
}

function logFraudEvent(msg,level='low'){
  const logContainer=document.getElementById('logEntries');
  const entry=document.createElement('div');
  entry.className='logEntry';
  const sev=document.createElement('span');
  sev.className='severity '+level;
  sev.textContent=level.toUpperCase();
  const text=document.createElement('span');
  text.textContent=new Date().toLocaleTimeString()+': '+msg;
  entry.appendChild(text);
  entry.appendChild(sev);
  logContainer.prepend(entry);
  visualAlert(level);
}
</script>
</body>
</html>
