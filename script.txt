// ============================== script.js ==============================

// BIEŻĄCY ROK
const yearEl = document.getElementById('year');
if (yearEl) yearEl.textContent = new Date().getFullYear();

// MENU MOBILNE
const mobileBtn = document.getElementById('mobile-btn');
const mobileNav = document.getElementById('mobile-nav');
mobileBtn?.addEventListener('click', () => mobileNav.classList.toggle('hidden'));
mobileNav?.querySelectorAll('a').forEach(a => a.addEventListener('click', () => mobileNav.classList.add('hidden')));

// I18N — ELEMENTY
const langPl = document.getElementById('lang-pl');
const langEn = document.getElementById('lang-en');
const langPlMobile = document.getElementById('lang-pl-mobile');
const langEnMobile = document.getElementById('lang-en-mobile');
const langNodes = document.querySelectorAll('[data-en]');
const langImgs  = document.querySelectorAll('img[data-alt-en]');

// ZAPISZ POLSKĄ WERSJĘ
langNodes.forEach(el => { if (!el.dataset.pl) el.dataset.pl = el.innerHTML; });
langImgs.forEach(img => { if (!img.dataset.altPl) img.dataset.altPl = img.alt; });

// FUNKCJA USTAWIANIA JĘZYKA
function setLanguage(lang) {
  if (lang === 'en') {
    langNodes.forEach(el => { el.innerHTML = el.dataset.en; });
    langImgs.forEach(img => { img.alt = img.dataset.altEn; });
    document.documentElement.lang = 'en';
    langEn?.classList.add('text-gold'); langPl?.classList.remove('text-gold');
    langEnMobile?.classList.add('text-gold'); langPlMobile?.classList.remove('text-gold');
  } else {
    langNodes.forEach(el => { el.innerHTML = el.dataset.pl; });
    langImgs.forEach(img => { img.alt = img.dataset.altPl; });
    document.documentElement.lang = 'pl';
    langPl?.classList.add('text-gold'); langEn?.classList.remove('text-gold');
    langPlMobile?.classList.add('text-gold'); langEnMobile?.classList.remove('text-gold');
  }
  // upewnij się, że rok jest ustawiony
  if (yearEl && !yearEl.textContent) yearEl.textContent = new Date().getFullYear();
  // pamiętaj wybór i aktualizuj URL
  localStorage.setItem('lang', lang);
  const sp = new URLSearchParams(location.search); sp.set('lang', lang);
  history.replaceState({}, '', `${location.pathname}?${sp.toString()}`);
}

// INICJALIZACJA JĘZYKA (URL -> localStorage -> przeglądarka)
(function initLang(){
  const urlLang = new URLSearchParams(location.search).get('lang');
  const stored = localStorage.getItem('lang');
  const browser = ((navigator.language || 'pl').toLowerCase().startsWith('en')) ? 'en' : 'pl';
  const initial = urlLang || stored || browser;
  setLanguage(initial);
})();

// ZDARZENIA JĘZYKOWE
langEn?.addEventListener('click', e => { e.preventDefault(); setLanguage('en'); });
langPl?.addEventListener('click', e => { e.preventDefault(); setLanguage('pl'); });
langEnMobile?.addEventListener('click', e => { e.preventDefault(); setLanguage('en'); mobileNav?.classList.add('hidden'); });
langPlMobile?.addEventListener('click', e => { e.preventDefault(); setLanguage('pl'); mobileNav?.classList.add('hidden'); });

// REZERWACJE — ELEMENTY
const reserveForm = document.getElementById('reserve-form');
const reserveOk   = document.getElementById('reserve-ok');
const reserveErr  = document.getElementById('reserve-err');

// DATA MINIMALNA = DZIŚ
const dateInput = reserveForm?.querySelector('input[name="date"]');
if (dateInput) {
  const t = new Date(); t.setHours(0,0,0,0);
  const y = t.getFullYear(), m = String(t.getMonth()+1).padStart(2,'0'), d = String(t.getDate()).padStart(2,'0');
  dateInput.min = `${y}-${m}-${d}`;
}

// POMOCNICZE
function showErr(msgPl, msgEn) {
  if (!reserveErr || !reserveOk) return;
  reserveErr.textContent = (document.documentElement.lang === 'en') ? (msgEn || 'Oops, check the fields and try again.') : (msgPl || 'Ups, sprawdź pola i spróbuj ponownie.');
  reserveErr.classList.remove('hidden');
  reserveOk.classList.add('hidden');
}
function showOk(id) {
  if (!reserveErr || !reserveOk) return;
  const msgPl = `Dziękujemy! Numer rezerwacji: ${id || '—'}`;
  const msgEn = `Thank you! Your reservation ID: ${id || '—'}`;
  reserveOk.textContent = (document.documentElement.lang === 'en') ? msgEn : msgPl;
  reserveOk.classList.remove('hidden');
  reserveErr.classList.add('hidden');
}

// WALIDACJA + FETCH
reserveForm?.addEventListener('submit', async (e) => {
  e.preventDefault();
  const fd = new FormData(reserveForm);
  if (fd.get('company')) return; // honeypot
  const phone = String(fd.get('phone')||'').trim();
  const email = String(fd.get('email')||'').trim();
  if (!/^[0-9+ ()-]{6,}$/.test(phone)) { showErr('Podaj poprawny numer telefonu.','Please enter a valid phone number.'); return; }
  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) { showErr('Podaj poprawny adres e-mail.','Please enter a valid email.'); return; }

  const payload = Object.fromEntries(fd.entries());
  payload.source = 'web';
  payload.lang = document.documentElement.lang || 'pl';

  try {
    const res = await fetch('/api/reservations', {
      method: 'POST',
      headers: {'Content-Type':'application/json'},
      body: JSON.stringify(payload)
    });
    if (!res.ok) throw new Error(await res.text());
    const json = await res.json();
    showOk(json.id);
    reserveForm.reset();
  } catch {
    showErr();
  }
});

// WEB VITALS -> /api/metrics (opcjonalne)
window.addEventListener('load', () => {
  if (!('webVitals' in window)) return;
  const send = (name, value) => {
    const body = JSON.stringify({ name, value, ts: Date.now() });
    navigator.sendBeacon?.('/api/metrics', body) || fetch('/api/metrics',{method:'POST',headers:{'Content-Type':'application/json'},body});
  };
  webVitals.onLCP(({value}) => send('LCP', value));
  webVitals.onCLS(({value}) => send('CLS', value));
  webVitals.onFID(({value}) => send('FID', value));
});

// SERVICE WORKER (PWA)
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js').catch(()=>{});
}
