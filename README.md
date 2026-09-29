ROLE
You are a senior full-stack engineer, security-minded backend developer, and product designer. Build a production-quality, demo-ready web application for an 8-hour AI hackathon. Work in PLANNING MODE: first produce an implementation plan and task list as artifacts, then build in the order given, then verify everything in the browser and capture screenshots as proof.

REFERENCE
I attached "AirSentinel – Urban Air Quality & Pollution Alert System" (single HTML file). It is the approved prototype. Keep its dark glassmorphism look, its tabs, and all its features. Rebuild it as a proper full-stack app and add authentication, user accounts, an admin panel, a real database, and live data.

PROJECT
AirSentinel is a smart urban air-quality monitoring and alert platform. It collects air-quality and weather data, calculates AQI, detects pollution hotspots, analyzes weather impact, predicts short-term trends, and sends personalized location-based alerts with recommendations. (Hackathon challenge PS-02: Urban Air Quality & Pollution Alert System.)

TECH STACK (fast, reliable, zero external setup)
- Frontend: React + Vite + TypeScript, Tailwind CSS, React Router, Zustand, Recharts, Framer Motion (only for meaningful transitions), Lucide icons. Map: custom canvas/SVG heatmap (as in the prototype), optional Leaflet.
- Backend: Node.js + Express + TypeScript, SQLite via better-sqlite3 (file DB, no install), zod validation, node-cron, Server-Sent Events.
- Auth: email + password, bcrypt hashing, JWT in httpOnly + SameSite cookies, refresh tokens stored hashed in the DB.
- One command to run everything: `npm install && npm run seed && npm run dev` (concurrently). Include a `.env.example`; never hard-code secrets.

1. AUTHENTICATION AND LOGIN PAGE (new, top priority)
Pages: /login, /register, /forgot-password (simulated reset flow with an on-screen token, no real email needed), /onboarding, and protected app routes. Unauthenticated users are redirected to /login and returned to their page after sign-in.
Login page design (premium, split-screen):
- Left panel: animated live "AQI ticker" showing 8 cities with colored AQI chips, a large tagline ("Breathe smarter. Know before you go."), and 3 short benefit points. Subtle animated pollution-particle background that respects prefers-reduced-motion.
- Right panel: glass card with email, password (show/hide toggle), "Remember me", "Forgot password?", primary "Sign in" button, link to "Create account", and two "Demo access" buttons: "Continue as Citizen" and "Continue as Admin" (for judges).
- Inline validation, accessible labels, visible focus, error messages that say what went wrong and how to fix it, a loading state on the button, and a lockout message after 5 failed attempts in 10 minutes (rate limiting).
- Fully responsive (375px to desktop) and works in dark and light themes.
Register: name, email, password (strength meter, min 8 chars), confirm password, and terms checkbox.
Onboarding (3 steps): choose home city (searchable, grouped by region), optionally use GPS or enter coordinates, then pick health profile (General, Sensitive group, Athlete) and alert threshold. Save to the user profile.
Roles: citizen, analyst, admin. Role-based route guards on both frontend and backend.
Security: helmet, CORS allow-list, express-rate-limit on auth routes, zod validation on every input, parameterized SQL only, password never logged or returned, CSRF-safe cookie settings, generic error messages for failed logins, logout that revokes the refresh token.
Seed demo accounts (documented in README): citizen@demo.com, analyst@demo.com, admin@demo.com (password Demo@1234).

2. DATABASE (SQLite)
users(id, name, email UNIQUE, password_hash, role, health_profile, home_city_id, created_at, last_login)
refresh_tokens(id, user_id, token_hash, expires_at, revoked)
cities(id, name, country, region, lat, lon, base_pm25, base_temp, base_humidity)
locations(id, city_id, name, lat, lon, zone_type)  -- 25 grid areas per city
saved_locations(id, user_id, label, lat, lon, city_id)  -- includes custom "My location" coordinates
air_readings(id, location_id, ts, pm25, pm10, co, no2, aqi, dominant_pollutant, source, quality_flag)  UNIQUE(location_id, ts, source)
weather_readings(id, location_id, ts, temperature, humidity, wind_speed, wind_dir, pressure)
forecasts(id, location_id, generated_at, target_ts, pred_aqi, lower, upper, model_version)
hotspots(id, city_id, detected_at, location_id, aqi, dominant_pollutant)
alert_rules(id, user_id, city_id, threshold, level, channel, enabled)
alerts(id, user_id NULL, city_id, location_id, ts, level, aqi, message, recommendation, acknowledged)
model_runs(id, run_at, mae, rmse, horizon_hours, model_version)
ingestion_logs(id, run_at, source, rows_inserted, rows_rejected, status, error)
audit_logs(id, user_id, action, ts, meta)
Indexes on (location_id, ts), (city_id, ts), users.email.

3. DATASET AND DATA PIPELINE
- Cities: at least 44 worldwide cities grouped by region (India, Asia & Middle East, Europe, Americas, Africa & Oceania). Include many Indian cities (Delhi, Mumbai, Chennai, Bengaluru, Kolkata, Hyderabad, Pune, Ahmedabad, Jaipur, Lucknow, Kanpur, Patna, Coimbatore, Madurai, Tiruchirappalli, Karur) plus global cities like London, Paris, Tokyo, Beijing, New York, Los Angeles, Sydney, Dubai, Cairo, and more.
- Custom location: any user can add coordinates through GPS or manual entry, saved to their account, with the pollution baseline modeled from the nearest city (state this clearly in the UI).
- `npm run seed`: generate 30 days of hourly data for all cities x 25 areas using a seeded random generator: rush-hour peaks (about 9 AM and 7 PM), weekend dips, zone effects (traffic 1.25, industrial 1.45, residential 1.0, green 0.6, commercial 1.1), wind lowers and humidity raises PM2.5, and a few pollution episodes. Mark source="seed".
- Live ingestion: node-cron every 15 minutes calls Open-Meteo Air Quality API (pm2_5, pm10, carbon_monoxide, nitrogen_dioxide, us_aqi) and Open-Meteo Weather API (temperature_2m, relative_humidity_2m, wind_speed_10m, wind_direction_10m, pressure_msl) for the selected/active cities (batch coordinates per request). No API key needed. If a call fails, keep serving stored data with a "cached" badge and log the failure.
- Cleaning pipeline on all incoming data: drop impossible values, flag outliers (z-score > 3), interpolate short gaps, reject duplicates, log rows_inserted and rows_rejected.
- AQI: US EPA breakpoint formula for PM2.5, PM10, CO (ppm), NO2 (ppb); overall AQI = highest sub-index; store the dominant pollutant. Categories: Good, Moderate, Unhealthy for Sensitive Groups, Unhealthy, Very Unhealthy, Hazardous with the standard colors.
- Write /docs/DATASET.md: data dictionary, units, valid ranges, sources, cleaning rules, limitations.

4. FEATURES (keep and upgrade everything from the prototype)
Personalized Command Center (home after login): greeting, home-city AQI gauge, live KPI strip (average, worst area, best area, active alerts, data freshness), parameter cards with sparklines and trend arrows, top-5 hotspot list, health recommendation tuned to the user's health profile, and a 12-hour forecast strip.
All Cities: ranked table of every city by current AQI with region, dominant pollutant, PM2.5, and hotspot count; click to open. Add region filter, search, and compare 2-3 cities side by side.
Hotspot Map: 5x5 heatmap per city with a layer switch (AQI, PM2.5, PM10, NO2, CO), clickable cells with details, and the hotspot rule (AQI above city mean + 1 standard deviation, or above 150).
Forecast: 72-hour history + 12-24 hour prediction with a shaded confidence band, trend badge (Improving / Stable / Worsening), peak forecast, back-tested MAE/RMSE stored in model_runs, and a reliability score. Model: seasonal hour-of-day profile + decaying momentum, optionally blended with linear regression on wind and humidity. Refresh hourly from the DB.
Weather Impact: correlation of PM2.5 with wind, humidity, temperature; scatter plots; plain-language insights; wind-drift direction.
Alerts Center: per-user alert rules (city, threshold, level), live alert feed via SSE, toasts, optional browser notifications, acknowledge button, history, forecast-crossing alerts, and health recommendations by category and health profile.
Data Explorer: filters (city, zone, date range, pollutant, quality flag), sorting, pagination, and real CSV export (server-generated download). Data Quality panel: row counts, date range, % missing, flagged outliers, last ingestion time and status.
My Locations: saved places, GPS button, manual coordinates, rename, delete.
Profile and Settings: edit name, home city, health profile, threshold, theme (light/dark/system), change password, sign out of all devices.
Admin Panel (admin role only): user list with role change and disable/enable, ingestion health (logs, rejected rows, retry button), model performance history, system-wide alert threshold, "Broadcast alert" to all users, "Simulate pollution event" for a chosen city, and an audit log viewer.
Analyst role: read-only analytics plus data export; no user management.
Presentation mode: full-screen auto-rotating view for the demo.
Report: "Download report" (printable HTML/PDF) with city AQI, hotspots, forecast, and recommendations.

5. REST API (all under /api, JSON errors, zod validated, paginated where large)
Auth: POST /auth/register, /auth/login, /auth/logout, /auth/refresh, /auth/forgot, /auth/reset; GET /auth/me
Users: GET/PATCH /users/me, PATCH /users/me/password; admin: GET /admin/users, PATCH /admin/users/:id
Data: GET /cities, /locations?city=, /readings/latest?city=, /readings?location=&from=&to=&interval=, /hotspots?city=, /forecast?location=, /weather-impact?city=, /rankings
Alerts: GET /alerts, POST /alerts/:id/ack, GET/POST/PATCH/DELETE /alert-rules, POST /admin/broadcast, POST /admin/simulate-event
Places: GET/POST/DELETE /saved-locations
Data ops: GET /data/quality, GET /data/export.csv, GET /admin/ingestion-logs
Realtime: GET /stream (SSE, authenticated)

6. DESIGN SYSTEM
Dark glassmorphism with a cyan accent (as in the prototype), light theme supported. Inter for UI and a monospace font for numbers. Official AQI colors used consistently in the gauge, map, charts, and chips. Skeleton loaders, friendly empty and error states, toasts, keyboard navigation, WCAG-friendly contrast. Sentence-case, plain-language copy with specific button labels ("Sign in", "Save settings"). Cache API responses, debounce inputs, memoize heavy calculations so the dashboard feels instant.

7. CODE QUALITY
Clean structure: /client (pages, components, hooks, store, lib) and /server (routes, middleware, services, db, jobs, lib/aqi.ts, forecast.ts, cleaning.ts). Strong typing, small reusable components, comments on formulas and the model. Vitest tests for AQI, cleaning, forecast, hotspot detection, and auth (register, login, lockout, role guard). README with setup, architecture diagram (Mermaid), ER diagram, API list, security notes, demo accounts, and a 3-minute demo script.

BUILD ORDER (stop and verify after each step)
1. DB schema, seed, AQI + cleaning libs, tests
2. Auth API + login/register/onboarding pages + route guards
3. Data API + Command Center + All Cities + Hotspot Map
4. Forecast + Weather Impact
5. Alerts (rules, SSE, toasts) + My Locations + Profile
6. Data Explorer + CSV export + Data Quality
7. Admin panel + Presentation mode + Report
8. Live Open-Meteo ingestion with fallback, polish, README

ACCEPTANCE CHECKLIST (verify in the browser)
[ ] Clean clone runs with `npm install && npm run seed && npm run dev`
[ ] Login, logout, register, forgot password, and onboarding all work; protected routes redirect to /login
[ ] Wrong password shows a clear error; 5 failed attempts triggers the lockout message
[ ] Demo Citizen and Demo Admin buttons sign in instantly; Admin sees the Admin Panel, Citizen does not (backend also blocks it: 403)
[ ] 44+ cities available; custom GPS/manual location can be saved and opened
[ ] AQI values match manual calculation for 3 test inputs
[ ] Hotspots show on the map and in the ranked list; layer switch works
[ ] Forecast shows history, prediction, confidence band, and MAE from stored predictions
[ ] Alert rule triggers a live toast through SSE; "Simulate pollution event" creates alerts for the right users
[ ] CSV export downloads a real file; Data Quality shows real DB numbers
[ ] Live Open-Meteo data loads, and the app falls back to stored data with a "cached" badge when offline
[ ] No console errors; works at 375px and desktop; passwords are never visible in logs or API responses

FINAL OUTPUT
The complete working project plus: a short summary of what was built, run instructions, demo account list, and a 3-minute demo script that walks through: login -> personalized dashboard -> all cities -> hotspot map -> weather insight -> forecast -> live alert -> data explorer -> admin panel.
