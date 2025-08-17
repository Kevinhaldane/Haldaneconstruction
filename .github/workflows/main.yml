import React, { useEffect, useMemo, useState } from "react";
import { motion } from "framer-motion";
import { Clock, Download } from "lucide-react";
import Logo from "./assets/Picture1.png"; // Added company logo

/*
Enhancement: Employee log now records GPS coordinates (geo tags) at the time of shift start, break, and finish.
Added Google Maps integration to view shift locations.
Added Progressive Web App (PWA) support with manifest.json and service worker.
Added scheduled push notifications (8am clock-in reminder, 4pm clock-out reminder UK time).
Added serverless function trigger for daily email with shift data at 5pm UK time.
Added CSV export button for managers to download data anytime with proper headers.
Embedded company logo inside app header.
*/

// ---- Register Service Worker ----
if ("serviceWorker" in navigator) {
  window.addEventListener("load", () => {
    navigator.serviceWorker
      .register("/sw.js")
      .then((reg) => {
        console.log("Service Worker registered:", reg);
        Notification.requestPermission();
      })
      .catch((err) => console.error("Service Worker registration failed:", err));
  });
}

// ---- Utilities ----
const todayISO = () => new Date().toISOString().slice(0, 10);
const fmtTime = (ts) => (ts ? new Date(ts).toLocaleTimeString() : "—");
const fmtDate = (ts) => (ts ? new Date(ts).toLocaleDateString() : todayISO());

// ---- Local storage helpers ----
const LS_KEY = "construct_time_app_v1";
const loadStore = () => {
  try {
    const raw = localStorage.getItem(LS_KEY);
    return raw ? JSON.parse(raw) : null;
  } catch (e) {
    return null;
  }
};
const saveStore = (s) => localStorage.setItem(LS_KEY, JSON.stringify(s));

// ---- Google Maps component ----
function MapView({ lat, lng }) {
  if (!lat || !lng) return <span>—</span>;
  const src = `https://www.google.com/maps/embed/v1/view?key=YOUR_GOOGLE_MAPS_API_KEY&center=${lat},${lng}&zoom=16&maptype=roadmap`;
  return (
    <iframe
      title="map"
      width="200"
      height="150"
      loading="lazy"
      allowFullScreen
      src={src}
      className="rounded-xl border"
    />
  );
}

// ---- Push Notifications Scheduler ----
function scheduleDailyNotifications() {
  if (!("Notification" in window) || Notification.permission !== "granted") return;

  const now = new Date();
  const schedule = (hour, minute, title, body) => {
    const notifTime = new Date();
    notifTime.setHours(hour, minute, 0, 0);
    if (notifTime < now) notifTime.setDate(notifTime.getDate() + 1);
    const delay = notifTime.getTime() - now.getTime();
    setTimeout(() => {
      new Notification(title, { body });
      schedule(hour, minute, title, body); // reschedule for next day
    }, delay);
  };

  // UK Time adjustments (client local time assumed to be UK)
  schedule(8, 0, "Clock In Reminder", "Please clock in before 8am");
  schedule(16, 0, "Clock Out Reminder", "Don't forget to clock out by 4pm");
}

// ---- Trigger serverless function for daily email at 5pm UK ----
function scheduleDailyEmail(store) {
  const now = new Date();
  const sendTime = new Date();
  sendTime.setHours(17, 0, 0, 0); // 5pm UK
  if (sendTime < now) sendTime.setDate(sendTime.getDate() + 1);
  const delay = sendTime.getTime() - now.getTime();

  setTimeout(async () => {
    try {
      await fetch("/api/sendDailyReport", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ shifts: store.shifts, date: todayISO() }),
      });
      console.log("Daily report triggered");
    } catch (err) {
      console.error("Failed to trigger daily report", err);
    }
    scheduleDailyEmail(store); // reschedule for next day
  }, delay);
}

// ---- CSV Export Utility ----
function downloadCSV(shifts, employees) {
  const headers = [
    "First Name",
    "Last Name",
    "Date",
    "Time of Submission",
    "Log In Time",
    "Log Out Time",
    "Start Location",
    "Finish Location",
  ];

  const rows = shifts.map((s) => {
    const emp = employees.find((e) => e.id === s.user_id) || {};
    const [firstName, ...rest] = emp.name ? emp.name.split(" ") : ["", ""];
    const lastName = rest.join(" ");
    return [
      firstName,
      lastName,
      fmtDate(s.start_ts),
      fmtTime(s.start_ts),
      fmtTime(s.start_ts),
      fmtTime(s.finish_ts),
      s.start_geo ? `${s.start_geo.lat},${s.start_geo.lng}` : "",
      s.finish_geo ? `${s.finish_geo.lat},${s.finish_geo.lng}` : "",
    ].join(",");
  });

  const csv = [headers.join(","), ...rows].join("\n");
  const blob = new Blob([csv], { type: "text/csv" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `shifts_${todayISO()}.csv`;
  a.click();
  URL.revokeObjectURL(url);
}

// ---- Root component ----
export default function App() {
  const [store, setStore] = useState(
    () =>
      loadStore() || {
        employees: [
          { id: "u1", name: "Alex Mason", role: "worker" },
          { id: "u2", name: "Jordan Lee", role: "worker" },
        ],
        projects: [{ id: "p1", name: "Site A", address: "123 High St" }],
        shifts: [],
        plans: {},
        comms: [],
        acks: [],
        currentUserId: "u1",
      }
  );

  useEffect(() => saveStore(store), [store]);
  useEffect(() => {
    scheduleDailyNotifications();
    scheduleDailyEmail(store);
  }, [store]);

  const [tab, setTab] = useState("time");
  const currentUser = useMemo(
    () => store.employees.find((e) => e.id === store.currentUserId),
    [store]
  );

  return (
    <div className="min-h-screen bg-slate-50 text-slate-800">
      <header className="flex items-center justify-between p-4 bg-white shadow">
        <div className="flex items-center gap-3">
          <img src={Logo} alt="Haldane Construction" className="h-12" />
          <h1 className="text-xl font-bold text-red-700">
            Haldane Construction Services
          </h1>
        </div>
      </header>
      <main className="max-w-4xl mx-auto p-4 grid gap-4">
        {tab === "time" && (
          <TimeTab store={store} setStore={setStore} currentUser={currentUser} />
        )}
        <div className="flex justify-end">
          <button
            onClick={() => downloadCSV(store.shifts, store.employees)}
            className="flex items-center gap-2 px-3 py-2 rounded-xl border bg-blue-600 text-white shadow"
          >
            <Download className="w-4 h-4" /> Download CSV
          </button>
        </div>
      </main>
    </div>
  );
}

// ---- Time Tab ----
function TimeTab({ store, setStore, currentUser }) {
  const [projectId, setProjectId] = useState(store.projects[0]?.id || "");

  const openShift = store.shifts.find(
    (s) => s.user_id === currentUser.id && !s.finish_ts
  );

  const getGeo = async () => {
    return new Promise((resolve) => {
      if (!navigator.geolocation) return resolve(null);
      navigator.geolocation.getCurrentPosition(
        (pos) =>
          resolve({ lat: pos.coords.latitude, lng: pos.coords.longitude }),
        () => resolve(null),
        { enableHighAccuracy: true, timeout: 5000 }
      );
    });
  };

  const startShift = async () => {
    if (openShift) return;
    const now = new Date().toISOString();
    const geo = await getGeo();
    const shift = {
      id: `s_${Date.now()}`,
      user_id: currentUser.id,
      project_id: projectId || store.projects[0]?.id,
      date: todayISO(),
      start_ts: now,
      start_geo: geo,
      finish_ts: null,
      finish_geo: null,
      breaks: [],
      notes: "",
    };
    setStore({ ...store, shifts: [shift, ...store.shifts] });
  };

  const toggleBreak = async () => {
    if (!openShift) return;
    const last = openShift.breaks[openShift.breaks.length - 1];
    const now = new Date().toISOString();
    const geo = await getGeo();
    let updated;
    if (!last || last.end_ts) {
      updated = {
        ...openShift,
        breaks: [
          ...openShift.breaks,
          { start_ts: now, start_geo: geo, end_ts: null, end_geo: null },
        ],
      };
    } else {
      updated = {
        ...openShift,
        breaks: openShift.breaks.map((b, i) =>
          i === openShift.breaks.length - 1
            ? { ...b, end_ts: now, end_geo: geo }
            : b
        ),
      };
    }
    setStore({
      ...store,
      shifts: store.shifts.map((s) => (s.id === openShift.id ? updated : s)),
    });
  };

  const finishShift = async () => {
    if (!openShift) return;
    const now = new Date().toISOString();
    const geo = await getGeo();
    const last = openShift.breaks[openShift.breaks.length - 1];
    let breaks = openShift.breaks;
    if (last && !last.end_ts) {
      breaks = breaks.map((b, i) =>
        i === breaks.length - 1 ? { ...b, end_ts: now, end_geo: geo } : b
      );
    }
    const updated = {
      ...openShift,
      finish_ts: now,
      finish_geo: geo,
      breaks,
    };
    setStore({
      ...store,
      shifts: store.shifts.map((s) => (s.id === openShift.id ? updated : s)),
    });
  };

  const myShifts = store.shifts.filter((s) => s.user_id === currentUser.id);

  return (
    <section className="grid gap-4">
      <div className="bg-white rounded-2xl shadow p-4 grid gap-3">
        <h2 className="font-semibold text-lg flex items-center gap-2">
          <Clock className="w-5 h-5" /> Time Tracking
        </h2>
        <div className="flex flex-wrap items-center gap-2">
          <select
            className="border rounded-lg px-2 py-1"
            value={projectId}
            onChange={(e) => setProjectId(e.target.value)}
          >
            {store.projects.map((p) => (
              <option key={p.id} value={p.id}>
                {p.name}
              </option>
            ))}
          </select>
          <div className="ml-auto flex gap-2">
            <button
              onClick={startShift}
              disabled={!!openShift}
              className="px-3 py-2 rounded-xl border bg-green-600 text-white"
            >
              Start
            </button>
            <button
              onClick={toggleBreak}
              disabled={!openShift}
              className="px-3 py-2 rounded-xl border bg-amber-500 text-white"
            >
              Break
            </button>
            <button
              onClick={finishShift}
              disabled={!openShift}
              className="px-3 py-2 rounded-xl border bg-red-600 text-white"
            >
              Finish
            </button>
          </div>
        </div>
      </div>

      <div className="bg-white rounded-2xl shadow p-4 grid gap-3">
        <h3 className="font-semibold">My Shifts</h3>
        <table className="min-w-full text-sm">
          <thead>
            <tr className="bg-slate-100">
              <th className="p-2 text-left">Date</th>
              <th className="p-2 text-left">Start</th>
              <th className="p-2 text-left">Finish</th>
              <th className="p-2 text-left">Start Location</th>
              <th className="p-2 text-left">Finish Location</th>
            </tr>
          </thead>
          <tbody>
            {myShifts.map((s) => (
              <tr key={s.id} className="border-b align-top">
                <td className="p-2">{fmtDate(s.start_ts)}</td>
                <td className="p-2">{fmtTime(s.start_ts)}</td>
                <td className="p-2">{fmtTime(s.finish_ts)}</td>
                <td className="p-2">
                  {s.start_geo ? (
                    <MapView lat={s.start_geo.lat} lng={s.start_geo.lng} />
                  ) : (
                    "—"
                  )}
                </td>
                <td className="p-2">
                  {s.finish_geo ? (
                    <MapView lat={s.finish_geo.lat} lng={s.finish_geo.lng} />
                  ) : (
                    "—"
                  )}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </section>
  );
}
