---
layout: page
title: About
permalink: /about/
icon: fas fa-user
order: 1
---

<style>
.xbt-about {
  --ink:#ded6c6; --muted:#877b70; --gold:#d7b36a; --blood:#9b1c25;
  --border:#33201f; --panel:#100b0b; --panel2:#0c0909;
  font-family:'JetBrains Mono','Fira Code','SFMono-Regular',Consolas,monospace;
  color:var(--ink);
}
.xbt-about .sigil {
  padding:24px 18px; margin:0 0 24px;
  background:radial-gradient(circle at 80% 20%,rgba(155,28,37,.08),transparent 35%),linear-gradient(135deg,#100b0b,#080707);
  border:1px solid var(--border); border-left:3px solid var(--blood); border-radius:7px;
}
.xbt-about .sigil pre {
  margin:0; color:var(--gold); font-size:clamp(.5rem,1.6vw,.8rem);
  line-height:1.2; overflow-x:auto; text-shadow:0 0 9px rgba(215,179,106,.13);
}
.xbt-about .terminal,.xbt-about .card {
  background:var(--panel); border:1px solid var(--border); border-radius:7px;
}
.xbt-about .terminal { margin:18px 0 28px; overflow:hidden; }
.xbt-about .bar {
  padding:9px 13px; background:#0d0909; border-bottom:1px solid var(--border);
  color:#766a60; font-size:.65rem;
}
.xbt-about .bar b { color:var(--blood); }
.xbt-about .term-body { padding:16px; }
.xbt-about .prompt { color:var(--gold); }
.xbt-about .output { color:var(--muted); margin:4px 0 13px; line-height:1.6; }
.xbt-about .section { margin:30px 0; }
.xbt-about .section-title {
  display:flex; align-items:center; gap:8px; padding-bottom:8px;
  border-bottom:1px solid var(--border); color:var(--gold); font-size:.82rem;
}
.xbt-about .section-title::before { content:"†"; color:var(--blood); }
.xbt-about .section-title::after { content:" //"; color:#50302e; }
.xbt-about .intro { color:var(--muted); line-height:1.75; }
.xbt-about .quote {
  margin:24px 0; padding:16px 18px; border-left:3px solid var(--blood);
  background:rgba(155,28,37,.035); color:#a99d90; font-family:Georgia,serif; font-style:italic;
}
.xbt-about .cert-grid,.xbt-about .lab-grid {
  display:grid; grid-template-columns:repeat(2,minmax(0,1fr)); gap:11px; margin-top:14px;
}
.xbt-about .card { padding:14px; background:var(--panel2); }
.xbt-about .cert { border-left:2px solid #4a2b29; }
.xbt-about .cert.complete { border-left-color:var(--gold); }
.xbt-about .cert.progress { border-left-color:var(--blood); }
.xbt-about .cert-top { display:flex; justify-content:space-between; gap:10px; font-size:.72rem; }
.xbt-about .cert-name { color:var(--ink); font-weight:700; }
.xbt-about .complete .status { color:var(--gold); }
.xbt-about .progress .status { color:var(--blood); }
.xbt-about .bar-track { height:4px; margin:11px 0 8px; background:#211312; }
.xbt-about .bar-fill { display:block; height:100%; background:var(--gold); }
.xbt-about .progress .bar-fill { background:var(--blood); }
.xbt-about small { color:var(--muted); font-size:.63rem; }
.xbt-about .tag {
  display:inline-block; margin:4px 3px 4px 0; padding:5px 7px;
  color:#b8aa98; background:#100b0b; border:1px solid var(--border);
  border-radius:4px; font-size:.64rem;
}
.xbt-about .lab b { display:block; color:var(--gold); font-size:.72rem; }
.xbt-about .lab span { color:var(--muted); font-size:.67rem; }
.xbt-about .links a { color:var(--gold)!important; }
@media(max-width:700px){.xbt-about .cert-grid,.xbt-about .lab-grid{grid-template-columns:1fr;}}
</style>

<div class="xbt-about">

<div class="sigil">
<pre>
                 ╱╲
              ╱╲╱  ╲╱╲
             ╱  THE   ╲
            ╱ STRUGGLER ╲
            ╲     †     ╱
             ╲   ╱ ╲   ╱
              ╲╱   ╲╱
        ━━━━━━━━━━━━━━━━━━━━━
             0xBERK
        OFFENSIVE SECURITY
        RESEARCH // RED TEAM
</pre>
</div>

<div class="terminal">
  <div class="bar"><b>●</b> 0xberk // operator-profile // /about</div>
  <div class="term-body">
    <div><span class="prompt">root@0xberk:~$</span> whoami</div>
    <div class="output">António Rodrigues // Full Stack Developer → Offensive Security</div>
    <div><span class="prompt">root@0xberk:~$</span> cat /etc/mission</div>
    <div class="output">break systems to understand them. survive. adapt. learn.</div>
    <div><span class="prompt">root@0xberk:~$</span> cat /etc/focus</div>
    <div class="output">Active Directory attacks // internal network pentesting // red team operations</div>
  </div>
</div>

<div class="section">
  <div class="section-title">OPERATOR</div>
  <p class="intro">I'm <strong>António Rodrigues</strong> — Full Stack Developer transitioning into Offensive Security, with a focus on <strong>Active Directory attacks</strong>, <strong>internal network pentesting</strong>, and <strong>red team operations</strong>.</p>
  <p class="intro">My developer background gives me a unique perspective: I understand how systems are built, which makes it easier to find how they break.</p>
  <div class="quote">The struggler keeps moving. Every system has a path in. Every path teaches something.</div>
</div>

<div class="section" id="certifications">
  <div class="section-title">CERTIFICATION LEDGER</div>
  <div class="cert-grid">

    <div class="card cert complete">
      <div class="cert-top"><span class="cert-name">CPTS</span><span class="status">VERIFIED</span></div>
      <div class="bar-track"><span class="bar-fill" style="width:100%"></span></div>
      <small>Certified Penetration Testing Specialist // COMPLETE</small>
    </div>

    <div class="card cert progress">
      <div class="cert-top"><span class="cert-name">CRTO</span><span class="status">IN PROGRESS</span></div>
      <div class="bar-track"><span class="bar-fill" style="width:55%"></span></div>
      <small>Certified Red Team Operator // TRIAL IN PROGRESS</small>
    </div>

    <div class="card cert progress">
      <div class="cert-top"><span class="cert-name">OCP</span><span class="status">IN PROGRESS</span></div>
      <div class="bar-track"><span class="bar-fill" style="width:40%"></span></div>
      <small>Offensive security certification track // TRIAL IN PROGRESS</small>
    </div>

    <div class="card cert progress">
      <div class="cert-top"><span class="cert-name">ARTOC</span><span class="status">IN PROGRESS</span></div>
      <div class="bar-track"><span class="bar-fill" style="width:30%"></span></div>
      <small>Advanced Red Team Operator track // TRIAL IN PROGRESS</small>
    </div>

    <div class="card cert progress">
      <div class="cert-top"><span class="cert-name">CDSA</span><span class="status">IN PROGRESS</span></div>
      <div class="bar-track"><span class="bar-fill" style="width:25%"></span></div>
      <small>Certified Defensive Security Analyst // IN TRAINING</small>
    </div>

    <div class="card cert complete">
      <div class="cert-top"><span class="cert-name">CATO DISTINGUISHED SUPPORT ASSOCIATE</span><span class="status">VERIFIED</span></div>
      <div class="bar-track"><span class="bar-fill" style="width:100%"></span></div>
      <small>Completed</small>
    </div>

    <div class="card cert complete">
      <div class="cert-top"><span class="cert-name">CATO CERTIFIED ASSOCIATE</span><span class="status">VERIFIED</span></div>
      <div class="bar-track"><span class="bar-fill" style="width:100%"></span></div>
      <small>Completed</small>
    </div>

  </div>
</div>

<div class="section">
  <div class="section-title">HACK THE BOX // BATTLEGROUNDS</div>
  <div class="lab-grid">
    <div class="card lab"><b>DANTE</b><span>Pro Lab // conquered</span></div>
    <div class="card lab"><b>P.O.O.</b><span>Pro Lab // conquered</span></div>
    <div class="card lab"><b>100+</b><span>machines pwned</span></div>
  </div>
</div>

<div class="section">
  <div class="section-title">TECHNICAL FOCUS</div>
  <h3>Active Directory</h3>
  <div>
    <span class="tag">Kerberoasting</span><span class="tag">AS-REP Roasting</span>
    <span class="tag">NTLM Relay</span><span class="tag">ADCS Abuse</span>
    <span class="tag">Shadow Credentials</span><span class="tag">RBCD</span>
    <span class="tag">Domain Dominance</span>
  </div>
  <h3>General</h3>
  <div>
    <span class="tag">Privilege Escalation</span><span class="tag">Lateral Movement</span>
    <span class="tag">Pivoting</span><span class="tag">Attack Path Analysis</span>
  </div>
</div>

<div class="section">
  <div class="section-title">CURRENTLY STUDYING</div>
  <div class="lab-grid">
    <div class="card lab"><b>RED TEAM OPERATIONS</b><span>adversary tradecraft</span></div>
    <div class="card lab"><b>WEB APPLICATION SECURITY</b><span>attack surface analysis</span></div>
    <div class="card lab"><b>ADVERSARY SIMULATION</b><span>operator development</span></div>
    <div class="card lab"><b>SECURITY AUTOMATION</b><span>repeatable tooling</span></div>
  </div>
</div>

<div class="section links">
  <div class="section-title">RAVEN // LINKS</div>
  <ul>
    <li><a href="/assets/files/cv_antonio_rodrigues.pdf">Download CV</a></li>
    <li><a href="https://github.com/root4win">GitHub</a></li>
    <li><a href="https://www.linkedin.com/in/antónio-rodrigues-55996b2ab">LinkedIn</a></li>
    <li><a href="https://profile.hackthebox.com/profile/019c68bb-7975-7189-b36f-4294de0f7c9a">Hack The Box</a></li>
  </ul>
</div>

</div>
