
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>AMETLLA FIGHTERS</title>
<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
<style>
*{box-sizing:border-box}
body{margin:0;background:#101018;color:white;font-family:Arial,sans-serif;text-align:center}
header{background:#b51220;padding:22px;font-size:30px;font-weight:bold;letter-spacing:2px}
main{max-width:850px;margin:25px auto;padding:15px}
.panel{background:#20202c;border:1px solid #444;border-radius:14px;padding:20px;margin:15px 0}
h2{color:#ff4545}
button,select,input{padding:12px;border:0;border-radius:8px;margin:6px;font-size:15px}
button{background:#d51b2b;color:white;font-weight:bold;cursor:pointer}
button:hover{background:#ff3445}
input,select{background:#fff;color:#111}
#code{font-size:30px;color:#ff5353;font-weight:bold;letter-spacing:5px}
.teams{display:grid;grid-template-columns:1fr 1fr;gap:12px}
.team{background:#292936;border-radius:10px;padding:12px;min-height:130px}
.red{border-top:5px solid #f33}
.blue{border-top:5px solid #38aaff}
.player{background:#3b3b4b;padding:10px;margin:7px;border-radius:7px}
#status{color:#ffd45c;min-height:25px}
#start{width:85%;font-size:20px;padding:17px;background:#168c48}
#start:disabled{background:#555;cursor:not-allowed}
.hidden{display:none}
</style>
</head>
<body>
<header>⚔ AMETLLA FIGHTERS ⚔</header>
<main>

<div class="panel" id="menu">
<h2>VESTÍBULO ONLINE</h2>
<p>Elige el modo de juego</p>
<select id="mode">
<option value="1">1 VS 1</option>
<option value="2">2 VS 2</option>
<option value="3">3 VS 3</option>
<option value="4">4 VS 4</option>
</select>
<br>
<button onclick="createRoom()">CREAR SALA</button>
<hr>
<p>O introduce el código de una sala:</p>
<input id="joinCode" placeholder="CÓDIGO DE SALA" maxlength="6">
<button onclick="joinRoom()">UNIRSE</button>
</div>

<div class="panel hidden" id="lobby">
<h2>SALA DE JUEGO</h2>

<p>CÓDIGO DE LA SALA</p>
<div id="code">------</div>
<p id="modeText"></p>

<p>CAMBIAR NOMBRE</p>
<input id="username" placeholder="Escribe tu nombre" maxlength="16">
<button onclick="changeName()">GUARDAR NOMBRE</button>

<div class="teams">
<div class="team red">
<h3>🔴 EQUIPO ROJO</h3>
<div id="redPlayers"></div>
</div>
<div class="team blue">
<h3>🔵 EQUIPO AZUL</h3>
<div id="bluePlayers"></div>
</div>
</div>

<p id="status">Esperando jugadores...</p>
<button onclick="changeTeam()">CAMBIAR DE EQUIPO</button>
<br>
<button id="start" onclick="startGame()" disabled>INICIAR PARTIDA</button>
<br>
<button onclick="leaveRoom()">SALIR DE LA SALA</button>
</div>

<div class="panel hidden" id="game">
<h2>¡COMIENZA EL COMBATE!</h2>
<p id="gameInfo"></p>
<p>Esta versión muestra el inicio de la partida. El combate sincronizado se añadirá en la siguiente versión.</p>
<button onclick="leaveRoom()">VOLVER AL VESTÍBULO</button>
</div>

</main>

<script>
let peer=null,connections=[],host=false,roomCode="",myId="";
let players=[],maxTeam=1,myTeam=0,started=false;
let myUsername="";

function newId(){
 return Math.random().toString(36).substring(2,8).toUpperCase();
}

function connectPeer(id){
 return new Promise((resolve,reject)=>{
  peer=new Peer(id);
  peer.on("open",resolve);
  peer.on("error",reject);
 });
}

function createRoom(){
 maxTeam=Number(document.getElementById("mode").value);
 roomCode=newId();
 myId="JUGADOR-"+newId();
 myUsername="CREADOR";
 host=true;

 connectPeer("AMF-"+roomCode).then(()=>{
  peer.on("connection",conn=>{
   connections.push(conn);

   conn.on("data",data=>{
    if(data.type==="join"){
     if(players.length>=maxTeam*2 || started){
      conn.send({type:"full"});
      return;
     }

     players.push({
      id:data.id,
      name:data.name,
      team:data.team,
      conn:conn
     });

     broadcast();
     render();
    }

    if(data.type==="rename"){
     const player=players.find(p=>p.id===data.id);
     if(player){
      player.name=data.name;
      broadcast();
      render();
     }
    }
   });

   conn.on("close",()=>removeConnection(conn));
  });

  players=[{
   id:myId,
   name:myUsername,
   team:0,
   conn:null
  }];

  showLobby();
 }).catch(()=>{
  alert("No se pudo crear la sala. Puede que la red bloquee PeerJS.");
 });
}

function joinRoom(){
 let code=document.getElementById("joinCode").value.trim().toUpperCase();

 if(code.length!==6){
  alert("Escribe un código de 6 caracteres");
  return;
 }

 roomCode=code;
 host=false;
 myId="JUGADOR-"+newId();
 myUsername="JUGADOR";

 connectPeer().then(()=>{
  let conn=peer.connect("AMF-"+code);

  conn.on("open",()=>{
   connections=[conn];

   conn.send({
    type:"join",
    id:myId,
    name:myUsername,
    team:1
   });
  });

  conn.on("data",data=>{
   if(data.type==="full"){
    alert("La sala está llena");
    return;
   }

   if(data.type==="state"){
    players=data.players;
    maxTeam=data.maxTeam;
    started=data.started;
    render();
   }

   if(data.type==="start")showGame();
  });

  conn.on("error",()=>{
   alert("No se pudo conectar a esa sala.");
  });

  showLobby();
 }).catch(()=>{
  alert("No se pudo conectar. Comprueba el código y la conexión.");
 });
}

function showLobby(){
 document.getElementById("menu").classList.add("hidden");
 document.getElementById("lobby").classList.remove("hidden");
 document.getElementById("code").textContent=roomCode;
 document.getElementById("modeText").textContent=maxTeam+" VS "+maxTeam;
 document.getElementById("username").value=myUsername;
 render();
}

function changeName(){
 const name=document.getElementById("username").value.trim();

 if(!name){
  alert("Escribe un nombre");
  return;
 }

 myUsername=name.substring(0,16);

 if(host){
  const player=players.find(p=>p.id===myId);

  if(player){
   player.name=myUsername;
   broadcast();
   render();
  }
 }else{
  const conn=connections[0];

  if(conn&&conn.open){
   conn.send({
    type:"rename",
    id:myId,
    name:myUsername
   });
  }
 }
}

function broadcast(){
 let state={
  type:"state",
  players:players.map(p=>({
   id:p.id,
   name:p.name,
   team:p.team
  })),
  maxTeam,
  started
 };

 connections.forEach(c=>{
  if(c.open)c.send(state);
 });
}

function render(){
 let red=document.getElementById("redPlayers");
 let blue=document.getElementById("bluePlayers");

 red.innerHTML="";
 blue.innerHTML="";

 players.forEach(p=>{
  let el=document.createElement("div");
  el.className="player";
  el.textContent=p.name;

  (p.team===0?red:blue).appendChild(el);
 });

 let count=players.length;

 document.getElementById("status").textContent=
  count+" / "+(maxTeam*2)+" jugadores";

 document.getElementById("start").disabled=
  !host || count!==maxTeam*2;
}

function changeTeam(){
 let p=players.find(p=>p.id===myId);
 if(!p)return;

 p.team=p.team===0?1:0;

 if(host){
  broadcast();
 }else if(connections[0]?.open){
  connections[0].send({
   type:"team",
   id:myId,
   team:p.team
  });
 }

 render();
}

function startGame(){
 if(!host || players.length!==maxTeam*2)return;

 started=true;
 broadcast();

 connections.forEach(c=>{
  if(c.open)c.send({type:"start"});
 });

 showGame();
}

function showGame(){
 document.getElementById("lobby").classList.add("hidden");
 document.getElementById("game").classList.remove("hidden");
 document.getElementById("gameInfo").textContent=
  "Modo "+maxTeam+" VS "+maxTeam;
}

function removeConnection(conn){
 players=players.filter(p=>p.conn!==conn);
 connections=connections.filter(c=>c!==conn);
 broadcast();
 render();
}

function leaveRoom(){
 if(peer)peer.destroy();

 peer=null;
 connections=[];
 players=[];
 host=false;
 started=false;
 myUsername="";

 document.getElementById("game").classList.add("hidden");
 document.getElementById("lobby").classList.add("hidden");
 document.getElementById("menu").classList.remove("hidden");
}
</script>
</body>
</html>
