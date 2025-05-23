mport os
import zipfile

# Directory for assembling files
base_dir = '/mnt/data/dr_movie_all_files'
os.makedirs(base_dir, exist_ok=True)

# File contents
files = {
    '.gitignore': """# Ignore node modules and OS files
node_modules/
.DS_Store
""",
    'README.md': """# Dr. Movie

A static website prototype for **Dr. Movie**, an AI-powered mood-based movie recommendation service.

## Files

- `index.html` — Main page
- `style.css` — Branding styles (teal, burnt orange, soft gray)
- `script.js` — Vanilla JS for interactions & mock logic

## Getting Started

1. Clone this repo:
   ```bash
   git clone https://github.com/your-username/dr-movie.git
   cd dr-movie
   ```
2. Open `index.html` in your browser, or serve via a local static server:
   ```bash
   python -m http.server 8000
   ```
3. Browse to `http://localhost:8000`

## Next Steps

- Hook up real AI emotion-detection (Face API, Azure, etc.)
- Integrate TMDB or your backend recommendation API
- Add user profiles & persistence (Firebase, Express/Python API)
- Deploy via GitHub Pages, Netlify, Vercel, etc.
""",
    'index.html': """<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Dr. Movie</title>
  <link rel="stylesheet" href="style.css"/>
  <script defer src="script.js"></script>
</head>
<body>
  <header>
    <div class="container">
      <h1 class="logo">Dr. Movie</h1>
      <nav>
        <a href="#">Home</a>
        <a href="#">Features</a>
        <a href="#">About</a>
        <a href="#">Contact</a>
        <a href="#">Login</a>
      </nav>
    </div>
  </header>
  <main>
    <section class="hero">
      <h2>Your AI Doctor for Movie Therapy</h2>
      <button id="start-btn">Start My Diagnosis</button>
    </section>
    <section id="mood-input" class="hidden">
      <h3>How are you feeling?</h3>
      <div class="mood-buttons">
        <button data-mood="Happy">😊 Happy</button>
        <button data-mood="Sad">😢 Sad</button>
        <button data-mood="Anxious">😰 Anxious</button>
        <button data-mood="Bored">😒 Bored</button>
      </div>
      <button id="diagnose-btn" disabled>Diagnose</button>
    </section>
    <section id="prescription" class="hidden">
      <h3>Today's Mood Diagnosis</h3>
      <p id="diag-text"></p>
      <div class="card">
        <img id="movie-poster" src="" alt="Movie Poster"/>
        <div class="card-content">
          <h4 id="movie-title"></h4>
          <p id="movie-reason"></p>
          <a id="watch-link" href="#" target="_blank">Watch Now</a>
        </div>
      </div>
      <button id="save-btn">Save to Watchlist</button>
    </section>
    <section id="watchlist" class="hidden">
      <h3>My Watchlist</h3>
      <ul id="watchlist-items"></ul>
    </section>
  </main>
  <footer>
    <p>© 2025 Dr. Movie. All rights reserved.</p>
  </footer>
</body>
</html>
""",
    'style.css': """:root {
  --teal:       #008080;
  --orange:     #FF6600;
  --soft-gray:  #F2F2F2;
  --dark-text:  #333333;
  --white:      #FFFFFF;
}
* { box-sizing: border-box; margin:0; padding:0; }
body { font-family: sans-serif; color: var(--dark-text); background: var(--white); }
.container { display: flex; align-items: center; justify-content: space-between; max-width: 900px; margin: 0 auto; padding: 0.5rem; }
header { background: var(--teal); }
header .logo { color: var(--white); font-size: 1.5rem; }
header nav a { color: var(--white); margin-left: 1rem; text-decoration: none; }
.hero { border: 2px solid var(--teal); margin: 1rem auto; max-width: 900px; padding: 2rem; text-align: center; }
.hero button { background: var(--orange); color: var(--white); border: none; padding: 0.75rem 1.5rem; font-size: 1rem; cursor: pointer; border-radius: 4px; }
section { max-width: 900px; margin: 1rem auto; padding: 1rem; border: 1px solid var(--teal); border-radius: 4px; background: var(--white); }
.hidden { display: none; }
#mood-input { background: var(--soft-gray); }
#mood-input .mood-buttons button { margin: 0.5rem; padding: 0.5rem 1rem; border: 2px solid var(--orange); background: var(--white); cursor: pointer; border-radius: 4px; }
#mood-input .mood-buttons button.selected { background: var(--orange); color: var(--white); }
.card { display: flex; margin: 1rem 0; }
.card img { width: 120px; border-radius: 4px; }
.card-content { margin-left: 1rem; }
.card-content a { display: inline-block; margin-top: 0.5rem; background: var(--teal); color: var(--white); padding: 0.5rem 1rem; text-decoration: none; border-radius: 4px; }
#watchlist { background: var(--soft-gray); }
#watchlist-items li { margin: 0.5rem 0; list-style: none; display: flex; align-items: center; }
#watchlist-items img { width: 50px; margin-right: 0.5rem; }
footer { text-align: center; margin: 2rem 0; }
""",
    'script.js': """const startBtn     = document.getElementById('start-btn');
const moodSection  = document.getElementById('mood-input');
const presSection  = document.getElementById('prescription');
const watchSection = document.getElementById('watchlist');
const diagnoseBtn  = document.getElementById('diagnose-btn');
const moodButtons  = document.querySelectorAll('#mood-input .mood-buttons button');
const diagText     = document.getElementById('diag-text');
const titleEl      = document.getElementById('movie-title');
const posterEl     = document.getElementById('movie-poster');
const reasonEl     = document.getElementById('movie-reason');
const linkEl       = document.getElementById('watch-link');
const saveBtn      = document.getElementById('save-btn');
const watchlistEl  = document.getElementById('watchlist-items');
let selectedMood = '';
let watchlist = [];
startBtn.addEventListener('click', () => {
  moodSection.classList.remove('hidden');
  presSection.classList.add('hidden');
  watchSection.classList.add('hidden');
});
moodButtons.forEach(btn => {
  btn.addEventListener('click', () => {
    moodButtons.forEach(b => b.classList.remove('selected'));
    btn.classList.add('selected');
    selectedMood = btn.dataset.mood;
    diagnoseBtn.disabled = false;
  });
});
diagnoseBtn.addEventListener('click', () => {
  const mock = {
    Happy:   { title: "Singin' in the Rain", reason: "Keep that joy flowing!", poster: "https://via.placeholder.com/120", link: "#" },
    Sad:     { title: "Inside Out", reason: "A heartfelt journey to match your mood.", poster: "https://via.placeholder.com/120", link: "#" },
    Anxious: { title: "Finding Nemo", reason: "A calm adventure to soothe your nerves.", poster: "https://via.placeholder.com/120", link: "#" },
    Bored:   { title: "The Grand Budapest Hotel", reason: "A whimsical ride to spark your interest.", poster: "https://via.placeholder.com/120", link: "#" }
  };
  const pres = mock[selectedMood] || mock.Happy;
  diagText.textContent  = `You are feeling ${selectedMood}. We prescribe:`;
  titleEl.textContent   = pres.title;
  reasonEl.textContent  = pres.reason;
  posterEl.src          = pres.poster;
  linkEl.href           = pres.link;
  presSection.classList.remove('hidden');
  moodSection.classList.add('hidden');
  watchSection.classList.add('hidden');
});
saveBtn.addEventListener('click', () => {
  watchlist.push({ title: titleEl.textContent, poster: posterEl.src });
  renderWatchlist();
  watchSection.classList.remove('hidden');
});
function renderWatchlist() {
  watchlistEl.innerHTML = '';
  watchlist.forEach(item => {
    const li = document.createElement('li');
    li.innerHTML = `<img src="${item.poster}" alt="poster"/> ${item.title}`;
    watchlistEl.appendChild(li);
  });
}"""
}

# Write files
for filename, content in files.items():
    path = os.path.join(base_dir, filename)
    with open(path, 'w') as f:
        f.write(content)

# Create ZIP
zip_path = '/mnt/data/dr_movie_all_files.zip'
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zf:
    for filename in files:
        zf.write(os.path.join(base_dir, filename), filename)

zip_path
