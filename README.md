<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Chat User - Online Tanpa Setup</title>
<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600;700&display=swap" rel="stylesheet">
<style>
*{font-family:'Poppins',sans-serif;box-sizing:border-box}
:root{--me:linear-gradient(135deg,#8e2de2,#4a00e0);--you:#2a2a35}
body{margin:0;min-height:100vh;background:radial-gradient(circle at top, #2a2a4a, #0f0f0f 70%);display:flex;justify-content:center;align-items:center;padding:20px}
.card{width:100%;max-width:380px;background:rgba(255,255,255,0.06);backdrop-filter:blur(20px);border:1px solid rgba(255,255,255,0.1);border-radius:24px;overflow:hidden}
#pageLogin{padding:30px 24px}
.input{width:100%;background:#1e1e2a;border:1px solid #2a2a35;border-radius:12px;padding:14px;color:#fff;outline:none;margin-bottom:12px}
.btn{width:100%;padding:14px;border:none;border-radius:12px;background:var(--me);color:#fff;font-weight:700;cursor:pointer}
#pageChat{display:none;flex-direction:column;height:85vh;max-height:650px}
.header{padding:14px 16px;background:rgba(0,0,0,0.2);display:flex;justify-content:space-between;align-items:center}
#chat{flex:1;overflow-y:auto;padding:16px;display:flex;flex-direction:column;gap:10px;align-items:center}
.bubble{max-width:85%;padding:10px 14px;border-radius:18px;font-size:14px;cursor:pointer}
.me{align-self:flex-end;background:var(--me);margin-left:auto;color:#fff}
.you{align-self:flex-start;background:var(--you);margin-right:auto;color:#fff}
.system{align-self:center;background:rgba(250,204,21,0.15);border:1px solid rgba(250,204,21,0.3);color:#facc15;font-size:11px;padding:6px 12px;border-radius:20px}
.footer{padding:12px;background:#12121a;display:flex;gap:8px}
.footer input{flex:1;background:#1e1e2a;border:1px solid #2a2a35;border-radius:12px;padding:12px;color:#fff;outline:none}
.footer button{padding:12px 16px;border:none;border-radius:12px;background:var(--me);color:#fff;font-weight:700;cursor:pointer}
#overlay{position:fixed;inset:0;background:rgba(0,0,0,0.7);display:none;align-items:center;justify-content:center;z-index:999}
#overlay.show{display:flex}
.modal{background:#1e1e2a;border:1px solid #333;border-radius:16px;padding:22px;width:90%;max-width:320px;text-align:center}
</style>
</head>
<body>
<div class="card">
  <div id="pageLogin">
    <div style="text-align:center;font-weight:700;font-size:20px">Chat Online</div>
    <div style="text-align:center;color:#aaa;font-size:11px;margin:6px 0 20px">Bisa dipakai teman lewat link yang sama<br><span style="color:#4ade80">● Online via ntfy.sh (tanpa daftar)</span></div>
    <input id="inputNama" class="input" placeholder="Nama kamu">
    <button class="btn" onclick="masuk()">Masuk</button>
  </div>
  <div id="pageChat">
    <div class="header"><div><b id="namaAktif">-</b><div style="font-size:11px;color:#4ade80">● terhubung online</div></div><button onclick="logout()" style="background:rgba(255,80,80,0.15);color:#ff6b6b;border:1px solid rgba(255,80,80,0.3);padding:6px 10px;border-radius:8px;font-size:11px;cursor:pointer">keluar</button></div>
    <div id="chat"></div>
    <div class="footer"><input id="pesan" placeholder="Ketik..."><button onclick="kirim()">Kirim</button></div>
  </div>
</div>
<div id="overlay" onclick="this.classList.remove('show')"><div class="modal" onclick="event.stopPropagation()"><h3>Hapus Pesan?</h3><button class="btn" style="background:#ff1a1a;margin-top:10px" onclick="confirmDelete()">Hapus</button></div></div>

<script>
// GANTI NAMA TOPIC INI BIAR CHAT LU PRIVAT - harus sama di File 1 & File 2
const TOPIC = "vortexsione_chat_2026_host_user"; // ganti jadi nama unik lu misal vortexsione-art-xyz123

let akunAktif=null, semuaPesan=[], deleteKey=null, es=null;

function masuk(){
  let nama=document.getElementById('inputNama').value.trim();
  if(!nama||nama.length<3){alert('Nama min 3 huruf');return;}
  akunAktif=nama; localStorage.setItem('akunAktif',nama);
  document.getElementById('pageLogin').style.display='none';
  document.getElementById('pageChat').style.display='flex';
  document.getElementById('namaAktif').innerText=nama;
  // load dari local biar ada history
  semuaPesan=JSON.parse(localStorage.getItem('pesan_'+TOPIC)||'[]');
  tampilkan();
  // connect ke ntfy.sh
  connectNtfy();
}

function connectNtfy(){
  if(es) es.close();
  // SSE listen
  es = new EventSource('https://ntfy.sh/'+TOPIC+'/sse');
  es.onmessage = (e)=>{
    try{
      let data=JSON.parse(e.data);
      if(!data.message) return;
      let msg=JSON.parse(data.message);
      // cegah duplikat
      if(semuaPesan.find(m=>m.id===msg.id)) return;
      semuaPesan.push(msg);
      localStorage.setItem('pesan_'+TOPIC, JSON.stringify(semuaPesan.slice(-200)));
      tampilkan();
    }catch(err){}
  };
}

function publish(msg){
  msg.id = Date.now()+'_'+Math.random().toString(36).substr(2,5);
  msg.timestamp = Date.now();
  fetch('https://ntfy.sh/'+TOPIC, {
    method:'POST',
    body: JSON.stringify(msg)
  });
  // tampilkan lokal dulu biar cepat
  semuaPesan.push(msg);
  localStorage.setItem('pesan_'+TOPIC, JSON.stringify(semuaPesan.slice(-200)));
  tampilkan();
}

function tampilkan(){
  let box=document.getElementById('chat'); box.innerHTML='';
  let filtered=semuaPesan.filter(m=>m.nama===akunAktif || m.to===akunAktif || m.to==='all' || m.from==='system');
  filtered.sort((a,b)=>a.timestamp-b.timestamp);
  filtered.forEach(item=>{
    if(item.deletedBy && item.deletedBy.includes(akunAktif)) return;
    if(item.from==='system'){
      box.innerHTML+=`<div style="width:100%;display:flex;justify-content:center"><div class="system">🔄 ${item.pesan}</div></div>`;
    } else {
      let isMe=item.from==='pengirim' && item.nama===akunAktif;
      let cls=isMe?'me':'you';
      box.innerHTML+=`<div style="width:100%;max-width:360px;display:flex;${isMe?'justify-content:flex-end':'justify-content:flex-start'}"><div class="bubble ${cls}" onclick="openDel('${item.id}')">${item.pesan}<div style="font-size:9px;opacity:.6">${item.jam}</div></div></div>`;
    }
  });
  box.scrollTop=box.scrollHeight;
}

function kirim(){
  let p=document.getElementById('pesan').value.trim(); if(!p) return;
  publish({
    nama: akunAktif,
    pesan: p,
    jam: new Date().toLocaleTimeString('id-ID',{hour:'2-digit',minute:'2-digit'}),
    from: 'pengirim',
    to: 'host',
    deletedBy: []
  });
  document.getElementById('pesan').value='';
}

function openDel(id){deleteKey=id; document.getElementById('overlay').classList.add('show');}
function confirmDelete(){
  let idx=semuaPesan.findIndex(m=>m.id===deleteKey);
  if(idx>-1){
    // soft delete untuk user ini
    if(!semuaPesan[idx].deletedBy) semuaPesan[idx].deletedBy=[];
    semuaPesan[idx].deletedBy.push(akunAktif);
    // publish event hapus
    publish({
      nama: 'system',
      from: 'system',
      type: 'delete',
      targetId: deleteKey,
      deleter: akunAktif,
      pesan: akunAktif+' menghapus pesan',
      jam: new Date().toLocaleTimeString('id-ID'),
      deletedBy: [],
      timestamp: Date.now(),
      id: Date.now()+'_del'
    });
    localStorage.setItem('pesan_'+TOPIC, JSON.stringify(semuaPesan.slice(-200)));
  }
  document.getElementById('overlay').classList.remove('show');
  tampilkan();
}
function logout(){akunAktif=null;localStorage.removeItem('akunAktif'); if(es) es.close(); document.getElementById('pageLogin').style.display='block'; document.getElementById('pageChat').style.display='none';}
document.getElementById('pesan').addEventListener('keydown',e=>{if(e.key==='Enter') kirim();});
window.onload=()=>{let s=localStorage.getItem('akunAktif'); if(s) document.getElementById('inputNama').value=s;}
</script>
</body>
</html>
