import { useState } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Select, SelectItem, SelectTrigger, SelectValue, SelectContent } from "@/components/ui/select";
import jsPDF from "jspdf";
import html2canvas from "html2canvas";

const EVENT_TYPES = {
  sales: { label: "Sales Call", color: "bg-blue-200", pipeline: ["Appointment Booked", "No Show", "Rescheduled", "Sat – No Sale", "Sold"] },
  review: { label: "Client Review", color: "bg-green-200", pipeline: ["Appointment Booked", "No Show", "Rescheduled", "Sat – No Sale", "Sold"] },
  recruiting: { label: "Recruiting", color: "bg-purple-200", pipeline: ["Interview Scheduled", "Interview Completed", "Hired", "Enrolled", "Onboarded"] },
  onboarding: { label: "Onboarding", color: "bg-yellow-200", pipeline: ["Interview Scheduled", "Interview Completed", "Hired", "Enrolled", "Onboarded"] }
};

export default function AgencyCalendarApp() {
  const todayKey = new Date().toISOString().slice(0, 10);
  const [selectedDate, setSelectedDate] = useState(todayKey);
  const [events, setEvents] = useState({});

  const [form, setForm] = useState({
    title: "",
    start: "",
    end: "",
    type: "sales",
    pipelineStage: EVENT_TYPES["sales"].pipeline[0],
    notes: "",
    applications: 0,
    apv: 0
  });

  const [metrics, setMetrics] = useState({
    calls: 0,
    contacts: 0,
    appointmentsBooked: 0,
    appointmentsSat: 0,
    interviews: 0,
    hired: 0,
    enrolled: 0,
    onboarded: 0,
    applications: 0,
    totalAPV: 0
  });

  // ---------- Event Handling ----------
  const addEvent = () => {
    if (!form.title || !form.start || !form.end) return;

    setEvents(prev => ({
      ...prev,
      [selectedDate]: [...(prev[selectedDate] || []), { ...form }]
    }));

    setMetrics(prev => ({
      ...prev,
      applications: prev.applications + Number(form.applications),
      totalAPV: prev.totalAPV + Number(form.apv)
    }));

    setForm({ title: "", start: "", end: "", type: form.type, pipelineStage: EVENT_TYPES[form.type].pipeline[0], notes: "", applications: 0, apv: 0 });
  };

  const updateMetric = (key, delta) => {
    setMetrics(prev => ({ ...prev, [key]: Math.max(0, prev[key] + delta) }));
  };

  const handleMetricChange = (key, value) => {
    const number = Number(value);
    if (!isNaN(number)) {
      setMetrics(prev => ({ ...prev, [key]: number }));
    }
  };

  // ---------- Custom Week (Fri → Thu) ----------
  const getCustomWeekDates = (dateStr) => {
    const date = new Date(dateStr);
    const day = date.getDay(); // 0=Sun, 5=Fri
    const diffToFriday = (day >= 5) ? day - 5 : day + 2;
    const friday = new Date(date);
    friday.setDate(date.getDate() - diffToFriday);

    const weekDates = [];
    for (let i = 0; i < 7; i++) {
      const d = new Date(friday);
      d.setDate(friday.getDate() + i);
      weekDates.push(d.toISOString().slice(0, 10));
    }
    return weekDates;
  };

  const weekDates = getCustomWeekDates(selectedDate);

  // ---------- Export Functions ----------
  const exportDailyCSV = () => {
    const dailyEvents = events[selectedDate] || [];
    if (!dailyEvents.length) return;

    const headers = ["Title", "Start", "End", "Type", "Pipeline Stage", "Notes", "Applications", "APV"];
    const rows = dailyEvents.map(evt => [
      evt.title,
      evt.start,
      evt.end,
      EVENT_TYPES[evt.type].label,
      evt.pipelineStage,
      evt.notes,
      evt.applications,
      evt.apv
    ]);

    const csvContent = "data:text/csv;charset=utf-8," +
      [headers.join(","), ...rows.map(r => r.join(","))].join("\n");

    const encodedUri = encodeURI(csvContent);
    const link = document.createElement("a");
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", `schedule-${selectedDate}.csv`);
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  const exportWeeklyCSV = () => {
    const weeklyEvents = weekDates.flatMap(d => (events[d] || []).map(evt => ({ ...evt, date: d })));
    if (!weeklyEvents.length) return;

    const headers = ["Date", "Title", "Start", "End", "Type", "Pipeline Stage", "Notes", "Applications", "APV"];
    const rows = weeklyEvents.map(evt => [
      evt.date,
      evt.title,
      evt.start,
      evt.end,
      EVENT_TYPES[evt.type].label,
      evt.pipelineStage,
      evt.notes,
      evt.applications,
      evt.apv
    ]);

    const csvContent = "data:text/csv;charset=utf-8," +
      [headers.join(","), ...rows.map(r => r.join(","))].join("\n");

    const encodedUri = encodeURI(csvContent);
    const link = document.createElement("a");
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", `weekly-schedule-${selectedDate}.csv`);
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  const exportWeeklyPDF = async () => {
    const container = document.getElementById("weekly-schedule");
    if (!container) return;

    const canvas = await html2canvas(container, { scale: 2 });
    const imgData = canvas.toDataURL("image/png");

    const pdf = new jsPDF("p", "pt", "a4");
    const pdfWidth = pdf.internal.pageSize.getWidth();
    const pdfHeight = (canvas.height * pdfWidth) / canvas.width;

    pdf.addImage(imgData, "PNG", 0, 0, pdfWidth, pdfHeight);
    pdf.save(`weekly-schedule-${selectedDate}.pdf`);
  };

  return (
    <div className="p-6 max-w-5xl mx-auto space-y-6">
      <h1 className="text-2xl font-semibold">Agency Weekly Planner</h1>

      {/* Date Selector */}
      <Card>
        <CardContent className="p-4 flex items-center gap-4">
          <Input type="date" value={selectedDate} onChange={e => setSelectedDate(e.target.value)} />
        </CardContent>
      </Card>

      {/* Metrics */}
      <Card>
        <CardContent className="p-4 grid grid-cols-2 md:grid-cols-3 gap-4">
          {Object.entries(metrics).map(([key, value]) => (
            <div key={key} className="flex flex-col border rounded p-2">
              <span className="capitalize mb-1">{key}</span>
              <div className="flex gap-2 mb-1">
                <Button size="sm" variant="outline" onClick={() => updateMetric(key, -1)}>-</Button>
                <Button size="sm" variant="outline" onClick={() => updateMetric(key, 1)}>+</Button>
              </div>
              <Input
                type="number"
                value={value}
                onChange={e => handleMetricChange(key, e.target.value)}
                placeholder={key === 'totalAPV' ? '$0' : '0'}
                className="text-sm"
              />
            </div>
          ))}
        </CardContent>
      </Card>

      {/* Add Event */}
      <Card>
        <CardContent className="p-4 space-y-3">
          <h2 className="font-semibold">Schedule Appointment</h2>
          <Input placeholder="Title / Client Name" value={form.title} onChange={e => setForm({ ...form, title: e.target.value })} />
          <div className="grid grid-cols-2 gap-2">
            <Input type="time" value={form.start} onChange={e => setForm({ ...form, start: e.target.value })} />
            <Input type="time" value={form.end} onChange={e => setForm({ ...form, end: e.target.value })} />
          </div>
          <p className="text-xs text-muted-foreground">Times display in 12-hour format (AM/PM) based on user device settings.</p>

          <Select value={form.type} onValueChange={val => setForm({ ...form, type: val, pipelineStage: EVENT_TYPES[val].pipeline[0] })}>
            <SelectTrigger>
              <SelectValue placeholder="Event Type" />
            </SelectTrigger>
            <SelectContent>
              {Object.entries(EVENT_TYPES).map(([key, val]) => (
                <SelectItem key={key} value={key}>{val.label}</SelectItem>
              ))}
            </SelectContent>
          </Select>

          <Select value={form.pipelineStage} onValueChange={val => setForm({ ...form, pipelineStage: val })}>
            <SelectTrigger>
              <SelectValue placeholder="Pipeline Stage" />
            </SelectTrigger>
            <SelectContent>
              {EVENT_TYPES[form.type].pipeline.map(stage => (
                <SelectItem key={stage} value={stage}>{stage}</SelectItem>
              ))}
            </SelectContent>
          </Select>

          <Input placeholder="Notes (optional)" value={form.notes} onChange={e => setForm({ ...form, notes: e.target.value })} />
          <Input type="number" placeholder="Applications Submitted" value={form.applications} onChange={e => setForm({ ...form, applications: e.target.value })} />
          <Input type="number" placeholder="APV ($)" value={form.apv} onChange={e => setForm({ ...form, apv: e.target.value })} />
          <Button onClick={addEvent}>Add Appointment</Button>
        </CardContent>
      </Card>

      {/* Export Buttons */}
      <div className="flex gap-2 flex-wrap">
        <Button onClick={() => window.print()}>Print Daily Schedule</Button>
        <Button onClick={exportDailyCSV}>Export Daily CSV</Button>
        <Button onClick={exportWeeklyCSV}>Export Weekly CSV</Button>
        <Button onClick={exportWeeklyPDF}>Export Weekly PDF (Canva Friendly)</Button>
      </div>

      {/* Daily Schedule */}
      <Card id="printable-schedule">
        <CardContent className="p-4 space-y-2">
          <h2 className="font-semibold">Daily Schedule</h2>
          {(events[selectedDate] || []).length === 0 && <p className="text-sm text-muted-foreground">No appointments scheduled.</p>}
          {(events[selectedDate] || []).map((evt, idx) => (
            <div key={idx} className={`p-2 rounded ${EVENT_TYPES[evt.type].color}`}>
              <div className="text-sm font-medium">{evt.title}</div>
              <div className="text-xs">{evt.start} – {evt.end} ({EVENT_TYPES[evt.type].label})</div>
              {evt.pipelineStage && <div className="text-xs font-medium">Pipeline: {evt.pipelineStage}</div>}
              {evt.notes && <div className="text-xs italic text-muted-foreground">Notes: {evt.notes}</div>}
              {evt.applications > 0 && <div className="text-xs">Applications: {evt.applications}</div>}
              {evt.apv > 0 && <div className="text-xs">APV: ${evt.apv}</div>}
            </div>
          ))}
        </CardContent>
      </Card>

      {/* Weekly Schedule */}
      <Card>
        <CardContent className="p-4 space-y-4">
          <h2 className="font-semibold">Weekly Schedule (Fri → Thu)</h2>
          {weekDates.map(d => {
            const isToday = d === todayKey;
            return (
              <div
                key={d}
                className={`border rounded p-2 ${isToday ? "bg-yellow-100 border-yellow-400" : ""}`}
              >
                <h3 className="font-medium mb-1">{d} {isToday && "(Today)"}</h3>
                {(events[d] || []).length === 0 && <p className="text-sm text-muted-foreground">No appointments.</p>}
                {(events[d] || []).map((evt, idx) => (
                  <div key={idx} className={`p-2 rounded mb-1 ${EVENT_TYPES[evt.type].color}`}>
                    <div className="text-sm font-medium">{evt.title}</div>
                    <div className="text-xs">{evt.start} – {evt.end} ({EVENT_TYPES[evt.type].label})</div>
                    {evt.pipelineStage && <div className="text-xs font-medium">Pipeline: {evt.pipelineStage}</div>}
                    {evt.notes && <div className="text-xs italic text-muted-foreground">Notes: {evt.notes}</div>}
                    {evt.applications > 0 && <div className="text-xs">Applications: {evt.applications}</div>}
                    {evt.apv > 0 && <div className="text-xs">APV: ${evt.apv}</div>}
                  </div>
                ))}
              </div>
            );
          })}
        </CardContent>
      </Card>

      {/* Hidden Weekly Schedule for PDF */}
      <div id="weekly-schedule" className="hidden">
        {weekDates.map(d => (
          <div key={d} className="mb-4">
            <h3 className="font-semibold">{d}</h3>
            {(events[d] || []).map((evt, idx) => (
              <div key={idx} className={`p-2 rounded ${EVENT_TYPES[evt.type].color} mb-1`}>
                <div className="text-sm font-medium">{evt.title}</div>
                <div className="text-xs">{evt.start} – {evt.end} ({EVENT_TYPES[evt.type].label})</div>
              </div>
            ))}
            {(events[d] || []).length === 0 && <p className="text-xs text-muted-foreground">No appointments.</p>}
          </div>
        ))}
      </div>

      {/* Print Styles */}
      <style>
        {`
          @media print {
            body * { visibility: hidden; }
            #printable-schedule, #printable-schedule * { visibility: visible; }
            #printable-schedule { position: absolute; left: 0; top: 0; width: 100%; }
          }
        `}
      </style>
    </div>
  );
}

