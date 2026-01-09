<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>VHub</title>
<style>
body{margin:0;min-height:100vh;font-family:Arial,sans-serif;transition:.3s}
.hidden{display:none}
button,input{padding:10px;border:none;border-radius:8px;font-size:14px}
button{background:#ff0000;color:#fff;font-weight:bold}
input{width:95%;margin:6px 0}
body.light{background:#fff;color:#000}
body.dark{background:#0f0f0f;color:#fff}
body.gradient{background:linear-gradient(135deg,#000428,#004e92,#6a0dad,#ff69b4);color:#fff}
.header{background:#111;padding:10px;display:flex;justify-content:space-between;align-items:center;position:sticky;top:0;z-index:5}
.grid{display:grid;grid-template-columns:1fr;gap:10px;padding:10px}
.card{background:#181818;border-radius:10px;overflow:hidden;padding:6px}
.card video{width:100%;aspect-ratio:16/9;object-fit:cover}
.card b{display:block;padding:6px}
.card .actions{display:flex;align-items:center;gap:5px;margin-top:5px}
.card .comments{margin-top:5px;padding:5px;background:#222;border-radius:6px}
.profile-box{padding:15px}
#create{position:fixed;bottom:15px;left:50%;transform:translateX(-50%);z-index:10}
</style>
</head>
<body class="dark">

<div id="auth">
  <h2 style="padding:10px">🎥 VHub</h2>
  <div style="padding:10px">
    <input id="pseudo" placeholder="Ton pseudo">
    <button onclick="login()">Entrer</button>
  </div>
</div>

<div id="home" class="hidden">
  <div class="header">
    <b>VHub</b>
    <div>
      <button onclick="openProfile()">Profil</button>
    </div>
  </div>
  <div id="videos" class="grid"></div>
</div>

<div id="profile" class="hidden">
  <div class="header">
    <button onclick="home()">⬅ Accueil</button>
    <b>Profil</b>
  </div>
  <div class="profile-box">
    <p id="pname"></p>
    <p id="pkey"></p>
    <h4>🎨 Thème</h4>
    <button onclick="setTheme('light')">🌞</button>
    <button onclick="setTheme('dark')">🌙</button>
    <button onclick="setTheme('gradient')">🌈</button>
    <br><br>
    <h4>Abonnements</h4>
    <button id="sub-btn" onclick="toggleSubscribe()">S’abonner / Se désabonner</button>
    <br><br>
    <button onclick="logout()">Déconnexion</button>
  </div>
</div>

<div id="create">
  <button onclick="openCreate()">➕ Créer</button>
</div>

<script>
const API_URL="https://vhub-backend.onrender.com"; // Remplace par ton URL Render
let user=null, privateKey=null, publicKey=null;
let subscriptions=[], videos=[];

function login(){
  const p=document.getElementById("pseudo").value.trim();
  if(!p){ alert("Pseudo requis"); return; }
  privateKey=Math.random().toString(36).slice(2);
  publicKey="VHUB256B-"+btoa(privateKey).slice(0,8);
  user={name:p,pk:publicKey};
  localStorage.setItem("vhub-user", JSON.stringify({user,privateKey}));
  home();
}
function logout(){ localStorage.clear(); location.reload(); }
function hideAll(){ ["auth","home","profile"].forEach(id=>document.getElementById(id).classList.add("hidden")); }
function home(){ hideAll(); document.getElementById("home").classList.remove("hidden"); loadVideos(); }
function openProfile(){
  hideAll();
  document.getElementById("profile").classList.remove("hidden");
  document.getElementById("pname").textContent="Pseudo : "+user.name;
  document.getElementById("pkey").textContent="Clé publique : "+publicKey;
  updateSubscribeButton();
}
function setTheme(t){ document.body.className=t; localStorage.setItem("theme",t); }
function toggleSubscribe(){
  const idx=subscriptions.indexOf(publicKey);
  if(idx>=0){ subscriptions.splice(idx,1); alert("Désabonné"); } 
  else { subscriptions.push(publicKey); alert("Abonné"); }
  updateSubscribeButton();
}
function updateSubscribeButton(){ document.getElementById("sub-btn").textContent=subscriptions.includes(publicKey)?"Se désabonner":"S’abonner"; }

function openCreate(){
  const input=document.createElement("input");
  input.type="file"; input.accept="video/*";
  input.onchange=async ()=>{
    const file=input.files[0];
    if(!file) return alert("Pas de fichier");
    const title=prompt("Titre de la vidéo ?");
    if(!title) return alert("Titre requis");
    const formData=new FormData();
    formData.append("video",file);
    formData.append("title",title);
    formData.append("uploader",user.name);
    try {
      const res=await fetch(`${API_URL}/upload`,{method:"POST",body:formData});
      if(res.ok){ alert("Vidéo uploadée !"); loadVideos(); }
      else alert("Erreur d'upload");
    } catch(e){ alert("Erreur : "+e.message); }
  };
  input.click();
}

function likeVideo(id){ fetch(`${API_URL}/like`,{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify({id})}).then(loadVideos); }
function dislikeVideo(id){ fetch(`${API_URL}/dislike`,{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify({id})}).then(loadVideos); }
function postComment(id){
  const text=document.getElementById("comment-"+id).value.trim();
  if(!text) return;
  fetch(`${API_URL}/comment`,{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify({id,user:user.name,text})}).then(loadVideos);
}

async function loadVideos(){
  try{
    const res=await fetch(`${API_URL}/videos`);
    const vids=await res.json();
    videos=vids;
    renderVideos();
  }catch(e){ console.error(e); }
}

function renderVideos(){
  const div=document.getElementById("videos");
  div.innerHTML="";
  videos.forEach(v=>{
    const d=document.createElement("div");
    d.className="card";
    let commentsHtml="";
    if(v.comments) v.comments.forEach(c=>commentsHtml+=`<div>${c.user}: ${c.text}</div>`);
    d.innerHTML=`
      <video src="${API_URL}/uploads/${v.filename}" controls></video>
      <b>${v.title}</b>
      <div class="actions">
        <button onclick="likeVideo(${v.id})">👍</button> <span>${v.likes||0}</span>
        <button onclick="dislikeVideo(${v.id})">👎</button> <span>${v.dislikes||0}</span>
      </div>
      <div class="comments">
        ${commentsHtml}
        <input type="text" id="comment-${v.id}" placeholder="Écrire un commentaire">
        <button onclick="postComment(${v.id})">Envoyer</button>
      </div>
    `;
    div.appendChild(d);
  });
}

const saved=JSON.parse(localStorage.getItem("vhub-user"));
if(saved){ user=saved.user; privateKey=saved.privateKey; publicKey=user.pk; home(); }
else document.getElementById("auth").classList.remove("hidden");
const th=localStorage.getItem("theme");
if(th) setTheme(th);
</script>

</body>
</html> 
