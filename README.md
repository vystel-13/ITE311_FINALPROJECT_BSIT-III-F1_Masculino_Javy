# Workout Log

**Student Name:** Masculino, Javy L.
**Course and Section:** ITE 311 – Web System and Technologies 2, BSIT III - F1

## Project Description

Workout Log is a web app for tracking gym workouts. Users log each exercise with its muscle group, sets, reps, and weight. The app then calculates total volume, heaviest lift, and personal bests, and tracks progress toward a weekly volume goal. All data is saved in the browser, and weights can be switched between kg and lb.

## Features

- Log exercises with sets, reps, and weight, with form validation
- Automatic personal best (PB) badges for the heaviest lift per exercise
- Total volume, total sets, and heaviest lift statistics
- Weekly volume goal with a progress bar
- Volume breakdown by muscle group
- Filter by muscle group and kg/lb unit toggle
- Search workouts by exercise name (`feature/search`)
- Optional notes on each exercise (`feature/notes`)
- Export the workout log to a CSV file (`feature/export-csv`)

## Technologies Used

- Laravel (PHP)
- Vue.js 3 (Composition API)
- Vite
- HTML, CSS, JavaScript
- Git and GitHub

## How to Run

```bash
composer install
npm install
copy .env.example .env
php artisan key:generate
php artisan serve
npm run dev
```

Then open http://127.0.0.1:8000 in your browser.