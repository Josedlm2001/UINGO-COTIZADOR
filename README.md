# UINGO-COTIZADOR
Quedate tranquilo,
moviendo con UINGO
Conecta con repartidores verificados. Tus clientes rastrean en tiempo real. Sin integraciones complejas. Solo crea pedidos y nosotros hacemos el resto.
// Supabase: Project Settings → API. La "anon key" es pública por diseño; la protegen las reglas RLS de schema.sql.
const SUPABASE_URL = "https://TU-PROYECTO.supabase.co";
const SUPABASE_ANON_KEY = "TU_ANON_KEY";
const WHATSAPP = "521XXXXXXXXXX"; // 52 + 1 + 10 dígitos
 
// Tarifas. Si las cambias aquí, cámbialas también en la función calcular_precio de schema.sql.
const KM_INCLUIDOS = 7;  // km incluidos en el precio base
const BLOQUE_KM = 2;     // cada bloque de 2 km extra (o fracción) cobra "extra"
const TARIFAS = {
  moto:      { nombre: "Moto",            base: 65,   extra: 20 },
  carro:     { nombre: "Carro",           base: 120,  extra: 20 },
  camioneta: { nombre: "Camioneta 1.2 t", base: 400,  extra: 100, cargador: 300 },
  camion:    { nombre: "Camión 3 t",      base: 1000, extra: 200, cargador: 300 },
};
(function () {
  "use strict";
  const $ = (s) => document.querySelector(s);
  const $$ = (s) => document.querySelectorAll(s);
  const db = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
  const dinero = (n) => Number(n).toLocaleString("es-MX", { style: "currency", currency: "MXN" });
  const msg = (sel, texto, error) => { const e = $(sel); e.textContent = texto || ""; e.classList.toggle("error", !!error); };
 
  let usuario = null, perfil = null, mapa = null, rutaCapa = null, km = null;
  const pts = { origen: null, destino: null };
  const marcadores = {};
 
  // ---------- Precio (misma fórmula que calcular_precio en schema.sql) ----------
  function precio(unidad, kms, cargadores) {
    const t = TARIFAS[unidad];
    const bloques = Math.max(0, Math.ceil((kms - KM_INCLUIDOS) / BLOQUE_KM));
    const carg = t.cargador ? cargadores * t.cargador : 0;
    return { bloques, extra: bloques * t.extra, carg, total: t.base + bloques * t.extra + carg };
  }
 
  // ---------- Acceso ----------
  $$("[data-auth]").forEach((b) => b.addEventListener("click", () => {
    $$("[data-auth]").forEach((x) => x.classList.toggle("activo", x === b));
    $("#form-login").hidden = b.dataset.auth !== "login";
    $("#form-registro").hidden = b.dataset.auth !== "registro";
    msg("#msg-auth", "");
  }));
  function alternarEmpresa(form) {
    const emp = form.querySelector('input[name="tipo"]:checked').value === "empresa";
    form.querySelector(".solo-empresa").hidden = !emp;
  }
  ["#form-registro", "#form-perfil"].forEach((s) => $(s).addEventListener("change", (e) => { if (e.target.name === "tipo") alternarEmpresa($(s)); }));
 
  $("#form-login").addEventListener("submit", async (e) => {
    e.preventDefault();
    const f = new FormData(e.target);
    const { error } = await db.auth.signInWithPassword({ email: f.get("email"), password: f.get("password") });
    if (error) msg("#msg-auth", "Correo o contraseña incorrectos.", true);
  });
  $("#form-registro").addEventListener("submit", async (e) => {
    e.preventDefault();
    const f = new FormData(e.target);
    const { data, error } = await db.auth.signUp({
      email: f.get("email"), password: f.get("password"),
      options: { data: { tipo: f.get("tipo"), nombre: f.get("nombre"), telefono: f.get("telefono"),
                         razon_social: f.get("razon_social") || null, rfc: f.get("rfc") || null } },
    });
    if (error) return msg("#msg-auth", error.message, true);
    if (!data.session) msg("#msg-auth", "Cuenta creada. Revisa tu correo para confirmarla y luego inicia sesión.");
  });
  $("#salir").addEventListener("click", () => db.auth.signOut());
 
  db.auth.onAuthStateChange((_evento, sesion) => {
    usuario = sesion ? sesion.user : null;
    setTimeout(usuario ? entrar : salir, 0);
  });
  function salir() {
    perfil = null;
    $("#vista-auth").hidden = false; $("#vista-app").hidden = true; $("#nav").hidden = true;
  }
  async function entrar() {
    $("#vista-auth").hidden = true; $("#vista-app").hidden = false; $("#nav").hidden = false;
    await cargarPerfil();
    verVista("cotizador");
  }
 
  // ---------- Navegación ----------
  function verVista(v) {
    ["cotizador", "historial", "perfil"].forEach((n) => { $("#sec-" + n).hidden = n !== v; });
    $$("#nav [data-vista]").forEach((b) => b.classList.toggle("activo", b.dataset.vista === v));
    if (v === "cotizador") iniciarMapa();
    if (v === "historial") cargarHistorial();
  }
  $$("#nav [data-vista]").forEach((b) => b.addEventListener("click", () => verVista(b.dataset.vista)));
 
  // ---------- Perfil ----------
  async function cargarPerfil() {
    const { data } = await db.from("perfiles").select("*").eq("id", usuario.id).single();
    perfil = data;
    if (!perfil) return;
    const f = $("#form-perfil");
    f.querySelector(`input[name="tipo"][value="${perfil.tipo}"]`).checked = true;
    f.nombre.value = perfil.nombre || ""; f.telefono.value = perfil.telefono || "";
    f.razon_social.value = perfil.razon_social || ""; f.rfc.value = perfil.rfc || "";
    alternarEmpresa(f);
    $("#saludo").textContent = `Hola, ${perfil.nombre}.`;
  }
  $("#form-perfil").addEventListener("submit", async (e) => {
    e.preventDefault();
    const f = new FormData(e.target);
    const emp = f.get("tipo") === "empresa";
    const { error } = await db.from("perfiles").update({
      tipo: f.get("tipo"), nombre: f.get("nombre"), telefono: f.get("telefono"),
      razon_social: emp ? f.get("razon_social") : null, rfc: emp ? f.get("rfc") : null,
    }).eq("id", usuario.id);
    msg("#msg-perfil", error ? "No se pudo guardar. Intenta de nuevo." : "Perfil actualizado.", !!error);
    if (!error) cargarPerfil();
  });
 
  // ---------- Mapa ----------
  function iniciarMapa() {
    if (mapa) return mapa.invalidateSize();
    mapa = L.map("mapa").setView([19.2433, -103.725], 12); // Colima
    L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", { maxZoom: 19, attribution: "© OpenStreetMap" }).addTo(mapa);
    mapa.on("click", (e) => poner($('input[name="modo"]:checked').value, e.latlng.lat, e.latlng.lng));
  }
  async function reversa(lat, lng) {
    try {
      const r = await (await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&accept-language=es&lat=${lat}&lon=${lng}`)).json();
      return r.display_name || null;
    } catch (e) { return null; }
  }
  async function poner(cual, lat, lng, texto) {
    pts[cual] = { lat, lng, txt: texto || `${lat.toFixed(5)}, ${lng.toFixed(5)}` };
    $("#txt-" + cual).value = pts[cual].txt;
    if (marcadores[cual]) marcadores[cual].setLatLng([lat, lng]);
    else {
      const icono = L.divIcon({ className: "", html: `<div class="pin pin-${cual}" style="width:28px;height:28px"><span>${cual === "origen" ? "A" : "B"}</span></div>`, iconSize: [28, 28], iconAnchor: [14, 28] });
      marcadores[cual] = L.marker([lat, lng], { icon: icono, draggable: true }).addTo(mapa);
      marcadores[cual].on("dragend", (e) => { const p = e.target.getLatLng(); poner(cual, p.lat, p.lng); });
    }
    if (cual === "origen" && !pts.destino) $('input[name="modo"][value="destino"]').checked = true;
    calcularRuta();
    if (!texto) { const t = await reversa(lat, lng); if (t && pts[cual].lat === lat) { pts[cual].txt = t; $("#txt-" + cual).value = t; } }
  }
  $$("[data-buscar]").forEach((b) => b.addEventListener("click", async () => {
    const cual = b.dataset.buscar, texto = $("#txt-" + cual).value.trim();
    if (!texto) return;
    msg("#msg-cot", "Buscando dirección…");
    try {
      const q = /colima/i.test(texto) ? texto : `${texto}, Colima, México`;
      const r = await (await fetch(`https://nominatim.openstreetmap.org/search?format=json&limit=1&countrycodes=mx&accept-language=es&q=${encodeURIComponent(q)}`)).json();
      if (!r.length) throw new Error();
      msg("#msg-cot", "");
      mapa.setView([r[0].lat, r[0].lon], 15);
      poner(cual, parseFloat(r[0].lat), parseFloat(r[0].lon), texto);
    } catch (e) { msg("#msg-cot", "No encontramos esa dirección. Agrega colonia o ciudad, o márcala en el mapa.", true); }
  }));
  async function calcularRuta() {
    if (!pts.origen || !pts.destino) return mostrar();
    try {
      const a = pts.origen, b = pts.destino;
      const d = await (await fetch(`https://router.project-osrm.org/route/v1/driving/${a.lng},${a.lat};${b.lng},${b.lat}?overview=full&geometries=geojson`)).json();
      if (d.code !== "Ok") throw new Error();
      km = Math.max(0.1, Math.ceil(d.routes[0].distance / 100) / 10);
      if (rutaCapa) mapa.removeLayer(rutaCapa);
      rutaCapa = L.geoJSON(d.routes[0].geometry, { style: { color: "#1450FF", weight: 5 } }).addTo(mapa);
      mapa.fitBounds(rutaCapa.getBounds(), { padding: [30, 30] });
      msg("#msg-cot", "");
    } catch (e) { km = null; msg("#msg-cot", "No se pudo calcular la ruta. Mueve los puntos e intenta de nuevo.", true); }
    mostrar();
  }
 
  // ---------- Unidades y precio en pantalla ----------
  Object.entries(TARIFAS).forEach(([clave, t], i) => {
    const l = document.createElement("label"); l.className = "opcion";
    l.innerHTML = '<input type="radio" name="unidad"><span class="titulo"></span><span class="detalle"></span>';
    const inp = l.querySelector("input"); inp.value = clave; inp.checked = i === 0;
    l.querySelector(".titulo").textContent = t.nombre;
    l.querySelector(".detalle").textContent = `${dinero(t.base)} hasta ${KM_INCLUIDOS} km, +${dinero(t.extra)} por cada ${BLOQUE_KM} km extra`;
    $("#unidades").append(l);
  });
  const unidad = () => $('input[name="unidad"]:checked').value;
  const cargadores = () => (TARIFAS[unidad()].cargador ? Math.min(5, Math.max(0, parseInt($("#cargadores").value, 10) || 0)) : 0);
 
  function mostrar() {
    $("#bloque-cargadores").hidden = !TARIFAS[unidad()].cargador;
    $("#whatsapp").hidden = true; $("#solicitar").disabled = false;
    if (!km) { $("#panel-vacio").hidden = false; $("#panel-resultado").hidden = true; return; }
    const t = TARIFAS[unidad()], p = precio(unidad(), km, cargadores());
    const lineas = [[`${t.nombre}, ${km.toFixed(1)} km`, t.base]];
    if (p.bloques) lineas.push([`${p.bloques} × ${BLOQUE_KM} km extra`, p.extra]);
    if (p.carg) lineas.push([`Cargadores extra (${cargadores()})`, p.carg]);
    $("#total").textContent = dinero(p.total);
    const ul = $("#desglose"); ul.replaceChildren();
    lineas.forEach(([c, m]) => {
      const li = document.createElement("li");
      const a = document.createElement("span"), b = document.createElement("span");
      a.textContent = c; b.textContent = dinero(m); li.append(a, b); ul.append(li);
    });
    $("#panel-vacio").hidden = true; $("#panel-resultado").hidden = false;
  }
  $("#unidades").addEventListener("change", mostrar);
  $("#cargadores").addEventListener("input", mostrar);
 
  // ---------- Solicitar ----------
  $("#solicitar").addEventListener("click", async () => {
    if (!km || !pts.origen || !pts.destino) return;
    $("#solicitar").disabled = true; msg("#msg-cot", "Enviando solicitud…");
    const { data, error } = await db.from("cotizaciones").insert({
      origen_txt: pts.origen.txt, origen_lat: pts.origen.lat, origen_lng: pts.origen.lng,
      destino_txt: pts.destino.txt, destino_lat: pts.destino.lat, destino_lng: pts.destino.lng,
      unidad: unidad(), km, cargadores: cargadores(),
    }).select().single();
    if (error) { $("#solicitar").disabled = false; return msg("#msg-cot", "No se pudo enviar la solicitud. Intenta de nuevo.", true); }
    msg("#msg-cot", `Solicitud enviada. Total confirmado: ${dinero(data.total)}.`);
    const texto = `Hola, soy ${perfil ? perfil.nombre : ""}. Solicité un servicio en Uingo (${TARIFAS[data.unidad].nombre}, ${data.km} km, ${dinero(data.total)}).\nRecolección: ${data.origen_txt}\nEntrega: ${data.destino_txt}`;
    $("#whatsapp").href = `https://wa.me/${WHATSAPP}?text=${encodeURIComponent(texto)}`;
    $("#whatsapp").hidden = false;
  });
 
  // ---------- Historial ----------
  async function cargarHistorial() {
    const ul = $("#lista"); ul.replaceChildren();
    const { data, error } = await db.from("cotizaciones").select("*").order("creado", { ascending: false }).limit(30);
    if (error || !data.length) { const li = document.createElement("li"); li.textContent = error ? "No se pudo cargar." : "Aún no tienes solicitudes."; ul.append(li); return; }
    data.forEach((c) => {
      const li = document.createElement("li");
      const fila = document.createElement("div"); fila.className = "fila";
      const a = document.createElement("span"), b = document.createElement("span");
      a.textContent = `${TARIFAS[c.unidad].nombre}, ${c.km} km`; b.textContent = dinero(c.total);
      fila.append(a, b);
      const s1 = document.createElement("small"); s1.textContent = `De: ${c.origen_txt}`;
      const s2 = document.createElement("small"); s2.textContent = `A: ${c.destino_txt}`;
      const s3 = document.createElement("small"); s3.textContent = new Date(c.creado).toLocaleString("es-MX");
      li.append(fila, s1, s2, s3); ul.append(li);
    });
  }
})();
