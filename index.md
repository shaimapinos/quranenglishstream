<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Audio Stream Player</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            overscroll-behavior-y: contain; /* Prevents pull-to-refresh in WebView */
        }
        /* Custom scrollbar for webkit browsers */
        .custom-scrollbar::-webkit-scrollbar {
            width: 6px;
        }
        .custom-scrollbar::-webkit-scrollbar-track {
            background: #2d3748; /* bg-gray-700 */
        }
        .custom-scrollbar::-webkit-scrollbar-thumb {
            background: #4a5568; /* bg-gray-600 */
            border-radius: 3px;
        }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover {
            background: #718096; /* bg-gray-500 */
        }
        /* For Firefox */
        .custom-scrollbar {
            scrollbar-width: thin;
            scrollbar-color: #4a5568 #2d3748;
        }

        .tab-active {
            border-bottom-width: 2px;
            border-color: #84cc16; /* Lime Green */
            color: #84cc16; /* Lime Green */
        }
        .modal {
            display: none; /* Hidden by default */
        }
        .modal.active {
            display: flex; /* Show when active */
        }
        /* Ensure player controls are easily tappable */
        .player-button {
            min-width: 44px;
            min-height: 44px;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .song-item {
            border-bottom-width: 1px;
            border-color: #4a5568; /* gray-600 */
        }
        .song-item:last-child {
            border-bottom-width: 0;
        }
        .song-item.selected {
            background-color: #4a5568; /* gray-600 */
        }
         /* Basic Toast Styling */
        .toast {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            background-color: #1f2937; /* bg-gray-800 */
            color: white;
            padding: 10px 20px;
            border-radius: 8px;
            z-index: 1000;
            opacity: 0;
            transition: opacity 0.3s ease-in-out, bottom 0.3s ease-in-out;
            visibility: hidden;
        }

        .toast.show {
            opacity: 1;
            bottom: 30px; /* Slide up */
            visibility: visible;
        }

        /* Now Playing Indicator Animation */
        @keyframes sound-wave {
          0%, 40%, 100% { transform: scaleY(0.4); }
          20% { transform: scaleY(1.0); }
        }
        .now-playing-indicator {
            display: flex;
            align-items: flex-end;
            height: 18px;
        }
        .now-playing-indicator span {
            display: inline-block;
            background-color: #84cc16;
            width: 3px;
            height: 100%;
            margin-right: 2px;
            animation: sound-wave 1.2s infinite ease-in-out;
            transform-origin: bottom;
        }
        .now-playing-indicator span:nth-child(2) { animation-delay: -0.2s; }
        .now-playing-indicator span:nth-child(3) { animation-delay: -0.4s; }
        .now-playing-indicator span:nth-child(4) { animation-delay: -0.6s; }

        /* Fade Animations for Tab Content */
        @keyframes fadeIn {
            from { opacity: 0; }
            to { opacity: 1; }
        }
        @keyframes fadeOut {
            from { opacity: 1; }
            to { opacity: 0; }
        }
        .fade-in {
            animation: fadeIn 0.3s ease-in-out forwards;
        }
        .fade-out {
            animation: fadeOut 0.3s ease-in-out forwards;
        }
        .timer-button-active {
            background-color: #84cc16 !important;
            color: #1f2937 !important;
        }
    </style>
</head>
<body class="bg-gray-900 text-white flex flex-col h-screen">

    <!-- Audio Element -->
    <audio id="audioPlayer"></audio>

    <!-- Top Bar: Current Song Info -->
    <div class="p-4 bg-gray-800 shadow-md relative">
        <div id="currentSongDisplay" class="text-center flex items-center justify-center">
             <div id="header-now-playing" class="now-playing-indicator hidden mr-2">
               <span></span><span></span><span></span><span></span>
            </div>
            <div>
                <p id="songTitle" class="text-lg font-semibold truncate">No Song Selected</p>
                <p id="songArtist" class="text-sm text-gray-400 truncate">---</p>
            </div>
        </div>
    </div>

    <!-- Progress Bar and Time -->
    <div class="p-4 bg-gray-800">
        <input type="range" id="progressBar" value="0" class="w-full h-2 bg-gray-700 rounded-lg appearance-none cursor-pointer accent-[#84cc16]">
        <div class="flex justify-between text-xs text-gray-400 mt-1">
            <span id="currentTime">0:00</span>
            <span id="duration">0:00</span>
        </div>
    </div>

    <!-- Player Controls -->
    <div class="p-4 bg-gray-800 flex items-center justify-around">
        <button id="shuffleBtn" class="player-button text-gray-400 hover:text-[#84cc16]"><i class="fas fa-random fa-lg"></i></button>
        <button id="prevBtn" class="player-button text-gray-300 hover:text-white"><i class="fas fa-step-backward fa-xl"></i></button>
        <button id="playPauseBtn" class="player-button text-[#84cc16] hover:text-[#65a30d] bg-gray-700 rounded-full w-16 h-16 flex items-center justify-center">
            <i class="fas fa-play fa-2x"></i>
        </button>
        <button id="nextBtn" class="player-button text-gray-300 hover:text-white"><i class="fas fa-step-forward fa-xl"></i></button>
        <button id="loopBtn" class="player-button text-gray-400 hover:text-[#84cc16]"><i class="fas fa-retweet fa-lg"></i></button>
    </div>

    <!-- Volume and Sleep Timer -->
    <div class="px-4 md:px-6 pt-2 pb-4 bg-gray-800 flex items-center justify-center relative">
        <!-- Centered Volume Controls -->
        <div class="flex items-center justify-center space-x-2 w-full">
            <i class="fas fa-volume-down text-gray-400"></i>
            <input type="range" id="volumeCtrl" min="0" max="1" step="0.01" value="0.5" class="w-1/2 md:w-1/3 h-1 bg-gray-700 rounded-lg appearance-none cursor-pointer accent-[#84cc16]">
            <i class="fas fa-volume-up text-gray-400"></i>
        </div>
        <!-- Sleep Timer Button absolutely positioned on the right -->
        <div class="absolute right-4 md:right-6">
             <button id="sleepTimerBtn" class="player-button text-gray-400 hover:text-[#84cc16]"><i class="fas fa-clock fa-lg"></i></button>
        </div>
    </div>


    <!-- Tabs for Song Lists -->
    <div class="flex border-b border-gray-700 sticky top-0 bg-gray-900 z-10">
        <button data-tab="library" class="tab-button flex-1 py-3 text-center text-gray-400 hover:text-white tab-active">Audio Library</button>
        <button data-tab="favorites" class="tab-button flex-1 py-3 text-center text-gray-400 hover:text-white">Favorites</button>
        <button data-tab="playlists" class="tab-button flex-1 py-3 text-center text-gray-400 hover:text-white">Playlists</button>
    </div>

    <!-- Song List Area -->
    <div id="songListContainer" class="flex-grow overflow-y-auto custom-scrollbar p-2">
        <!-- Library View -->
        <div id="libraryView" class="tab-content">
            <!-- Songs will be injected here -->
        </div>
        <!-- Favorites View -->
        <div id="favoritesView" class="tab-content hidden">
            <p class="text-gray-500 text-center p-4 hidden" id="noFavoritesMessage">No favorite Surahs yet.</p>
            <div id="favoriteSongsContainer">
                <!-- Favorite songs will be injected here -->
            </div>
        </div>
        <!-- Playlists View -->
        <div id="playlistsView" class="tab-content hidden">
            <div class="p-2">
                <button id="createPlaylistBtn" class="w-full bg-[#84cc16] hover:bg-[#65a30d] text-white font-semibold py-2 px-4 rounded-lg mb-2">
                    <i class="fas fa-plus-circle mr-2"></i>Create New Playlist
                </button>
                 <div id="myPlaylistsContainer">
                    <!-- Playlists will be listed here -->
                 </div>
                 <p class="text-gray-500 text-center p-4 hidden" id="noPlaylistsMessage">There are no playlists. Create one first!</p>
            </div>
        </div>
        <!-- Single Playlist Songs View -->
        <div id="singlePlaylistSongsView" class="tab-content hidden">
             <div class="flex items-center justify-between p-2 border-b border-gray-700">
                <button id="backToPlaylistsBtn" class="text-[#84cc16] hover:text-[#65a30d]">
                    <i class="fas fa-arrow-left mr-2"></i> Back to Playlists
                </button>
                <h3 id="currentPlaylistNameHeader" class="text-lg font-semibold">Playlist Surahs</h3>
            </div>
            <div id="songsInPlaylistContainer">
                <!-- Songs in the selected playlist will be injected here -->
            </div>
             <p class="text-gray-500 text-center p-4 hidden" id="noSongsInPlaylistMessage">This playlist is empty.</p>
        </div>
    </div>

    <!-- Sleep Timer Modal -->
    <div id="sleepTimerModal" class="modal fixed inset-0 bg-black bg-opacity-75 items-center justify-center z-50 p-4">
        <div class="bg-gray-800 p-6 rounded-lg shadow-xl w-full max-w-sm">
            <h3 id="sleepTimerTitle" class="text-xl font-semibold mb-4 text-center">Sleep Timer</h3>
            <div id="timerOptions" class="grid grid-cols-2 gap-3">
                <button data-minutes="15" class="bg-gray-700 hover:bg-[#84cc16] hover:text-gray-900 font-semibold py-3 px-4 rounded-lg">15 Minutes</button>
                <button data-minutes="30" class="bg-gray-700 hover:bg-[#84cc16] hover:text-gray-900 font-semibold py-3 px-4 rounded-lg">30 Minutes</button>
                <button data-minutes="60" class="bg-gray-700 hover:bg-[#84cc16] hover:text-gray-900 font-semibold py-3 px-4 rounded-lg">60 Minutes</button>
                <button data-minutes="120" class="bg-gray-700 hover:bg-[#84cc16] hover:text-gray-900 font-semibold py-3 px-4 rounded-lg">120 Minutes</button>
                <button data-minutes="0" class="col-span-2 bg-gray-700 hover:bg-pink-500 font-semibold py-3 px-4 rounded-lg">Turn Off</button>
            </div>
            <div class="flex justify-center mt-4">
                <button id="cancelSleepTimerBtn" class="bg-gray-600 hover:bg-gray-700 text-white font-semibold py-2 px-6 rounded-lg">Cancel</button>
            </div>
        </div>
    </div>


    <!-- Modal for Creating Playlist -->
    <div id="createPlaylistModal" class="modal fixed inset-0 bg-black bg-opacity-75 items-center justify-center z-50 p-4">
        <div class="bg-gray-800 p-6 rounded-lg shadow-xl w-full max-w-md">
            <h3 class="text-xl font-semibold mb-4">Create New Playlist</h3>
            <input type="text" id="newPlaylistName" placeholder="Playlist Name" class="w-full p-2 rounded bg-gray-700 border border-gray-600 focus:border-[#84cc16] outline-none mb-4">
            <div class="flex justify-end space-x-2">
                <button id="cancelCreatePlaylistBtn" class="bg-gray-600 hover:bg-gray-700 text-white font-semibold py-2 px-4 rounded-lg">Cancel</button>
                <button id="savePlaylistBtn" class="bg-[#84cc16] hover:bg-[#65a30d] text-white font-semibold py-2 px-4 rounded-lg">Create</button>
            </div>
        </div>
    </div>

    <!-- Modal for Adding to Playlist -->
    <div id="addToPlaylistModal" class="modal fixed inset-0 bg-black bg-opacity-75 items-center justify-center z-50 p-4">
        <div class="bg-gray-800 p-6 rounded-lg shadow-xl w-full max-w-md">
            <h3 class="text-xl font-semibold mb-4">Add to Playlist</h3>
            <input type="hidden" id="songIdToAddToPlaylist">
            <div id="playlistSelectionContainer" class="max-h-60 overflow-y-auto custom-scrollbar mb-4 border border-gray-700 rounded-md p-2">
                <!-- Available playlists for selection -->
            </div>
            <p id="noPlaylistsForAddingMessage" class="text-gray-400 mb-4 hidden">No playlists available. Create one first!</p>
            <div class="flex justify-end space-x-2">
                <button id="cancelAddToPlaylistBtn" class="bg-gray-600 hover:bg-gray-700 text-white font-semibold py-2 px-4 rounded-lg">Cancel</button>
            </div>
        </div>
    </div>
    
    <!-- NEW: Modal for Deleting Playlist -->
    <div id="deletePlaylistModal" class="modal fixed inset-0 bg-black bg-opacity-75 items-center justify-center z-50 p-4">
        <div class="relative bg-gray-800 rounded-2xl shadow-xl w-full max-w-md border border-gray-700">
            <div class="p-8 text-center">
                <div class="mx-auto flex items-center justify-center h-12 w-12 rounded-full bg-red-900/50 mb-4">
                    <svg class="h-6 w-6 text-red-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" d="M12 9v3.75m-9.303 3.376c-.866 1.5.217 3.374 1.948 3.374h14.71c1.73 0 2.813-1.874 1.948-3.374L13.949 3.378c-.866-1.5-3.032-1.5-3.898 0L2.697 16.126z" />
                    </svg>
                </div>
                <h3 class="text-xl font-bold text-white">Delete Playlist</h3>
                <p id="deleteModalText" class="mt-2 text-gray-400">
                    <!-- Confirmation message will be set here by JS -->
                </p>
            </div>
            <div class="flex justify-center items-center gap-4 bg-gray-800/50 p-4 rounded-b-2xl">
                <button id="cancelDeletePlaylistBtn" class="flex-1 rounded-lg bg-gray-700 hover:bg-gray-600 px-4 py-3 text-white font-semibold transition-colors duration-200">
                    Cancel
                </button>
                <button id="confirmDeletePlaylistBtn" class="flex-1 rounded-lg bg-red-600 hover:bg-red-700 px-4 py-3 text-white font-semibold transition-colors duration-200">
                    Delete
                </button>
            </div>
        </div>
    </div>


    <!-- Toast Notification -->
    <div id="toastNotification" class="toast">This is a toast message!</div>


    <script type="module">
        // Firebase Imports
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, addDoc, getDocs, arrayUnion, arrayRemove } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- Firebase Configuration ---
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {

            apiKey: "AIzaSyCKQhaLiswg-bGGKgnO5ScH2-xLzspToHA",
            authDomain: "quranenglishstream.firebaseapp.com",
            projectId: "quranenglishstream",
            storageBucket: "quranenglishstream.firebasestorage.app",
            messagingSenderId: "300392129839",
            appId: "1:300392129839:web:30b05802abeb2748f8edda",
            measurementId: "G-JX6YBY2DVM"

        };
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'audio-player-default-app';

        // Initialize Firebase
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);

        let userId = null;
        let dbUserDocRef = null;
        let dbPlaylistsCollectionRef = null;
        let unsubscribeUserDoc = null;
        let unsubscribePlaylists = null;


        // --- Audio Player State & Elements ---
        const audioPlayer = document.getElementById('audioPlayer');
        const playPauseBtn = document.getElementById('playPauseBtn');
        const prevBtn = document.getElementById('prevBtn');
        const nextBtn = document.getElementById('nextBtn');
        const loopBtn = document.getElementById('loopBtn');
        const shuffleBtn = document.getElementById('shuffleBtn');
        const progressBar = document.getElementById('progressBar');
        const currentTimeDisplay = document.getElementById('currentTime');
        const durationDisplay = document.getElementById('duration');
        const volumeCtrl = document.getElementById('volumeCtrl');
        const songTitleDisplay = document.getElementById('songTitle');
        const songArtistDisplay = document.getElementById('songArtist');
        const headerNowPlayingIndicator = document.getElementById('header-now-playing');

        const libraryView = document.getElementById('libraryView');
        const favoritesView = document.getElementById('favoritesView');
        const favoriteSongsContainer = document.getElementById('favoriteSongsContainer'); // <-- ADDED
        const playlistsView = document.getElementById('playlistsView');
        const singlePlaylistSongsView = document.getElementById('singlePlaylistSongsView');
        const songsInPlaylistContainer = document.getElementById('songsInPlaylistContainer');
        const currentPlaylistNameHeader = document.getElementById('currentPlaylistNameHeader');
        const backToPlaylistsBtn = document.getElementById('backToPlaylistsBtn');
        
        const noFavoritesMessage = document.getElementById('noFavoritesMessage');
        const noPlaylistsMessage = document.getElementById('noPlaylistsMessage');
        const myPlaylistsContainer = document.getElementById('myPlaylistsContainer');
        const noSongsInPlaylistMessage = document.getElementById('noSongsInPlaylistMessage');

        // Modals
        const createPlaylistModal = document.getElementById('createPlaylistModal');
        const newPlaylistNameInput = document.getElementById('newPlaylistName');
        const savePlaylistBtn = document.getElementById('savePlaylistBtn');
        const cancelCreatePlaylistBtn = document.getElementById('cancelCreatePlaylistBtn');
        const createPlaylistBtn = document.getElementById('createPlaylistBtn');

        const addToPlaylistModal = document.getElementById('addToPlaylistModal');
        const playlistSelectionContainer = document.getElementById('playlistSelectionContainer');
        const songIdToAddToPlaylistInput = document.getElementById('songIdToAddToPlaylist');
        const cancelAddToPlaylistBtn = document.getElementById('cancelAddToPlaylistBtn');
        const noPlaylistsForAddingMessage = document.getElementById('noPlaylistsForAddingMessage');
        
        // NEW: Delete Playlist Modal Elements
        const deletePlaylistModal = document.getElementById('deletePlaylistModal');
        const deleteModalText = document.getElementById('deleteModalText');
        const confirmDeletePlaylistBtn = document.getElementById('confirmDeletePlaylistBtn');
        const cancelDeletePlaylistBtn = document.getElementById('cancelDeletePlaylistBtn');

        const toastNotification = document.getElementById('toastNotification');
        
        // Sleep Timer Elements
        const sleepTimerBtn = document.getElementById('sleepTimerBtn');
        const sleepTimerModal = document.getElementById('sleepTimerModal');
        const sleepTimerTitle = document.getElementById('sleepTimerTitle');
        const timerOptions = document.getElementById('timerOptions');
        const cancelSleepTimerBtn = document.getElementById('cancelSleepTimerBtn');


        let songs = [
            { id: 's1', title: 'Surah 1: Al-Fatiha', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/001.mp3'},
            { id: 's2', title: 'Surah 2: Al-Baqarah', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/002.mp3'},
            { id: 's3', title: 'Surah 3: Al-Imran', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/003.mp3'},
            { id: 's4', title: 'Surah 4: An-Nisa', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/004.mp3'},
            { id: 's5', title: 'Surah 5: Al-Maeda', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/005.mp3'},
            { id: 's6', title: 'Surah 6: Al-Annam', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/006.mp3'},
            { id: 's7', title: 'Surah 7: Al-Araf', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/007.mp3'},
            { id: 's8', title: 'Surah 8: Al-Anfal', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/008.mp3'},
            { id: 's9', title: 'Surah 9: At-Tawba', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/009.mp3'},
            { id: 's10', title: 'Surah 10: Yunus', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/010.mp3'},
            { id: 's11', title: 'Surah 11: Hud', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/011.mp3'},
            { id: 's12', title: 'Surah 12: Yusuf', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/012.mp3'},
            { id: 's13', title: 'Surah 13: Ar-Rad', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/013.mp3'},
            { id: 's14', title: 'Surah 14: Ibrahim', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/014.mp3'},
            { id: 's15', title: 'Surah 15: Al-Hijr', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/015.mp3'},
            { id: 's16', title: 'Surah 16: An-Nahl', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/016.mp3'},
            { id: 's17', title: 'Surah 17: Al-Isra', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/017.mp3'},
            { id: 's18', title: 'Surah 18: Al-Kahf', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/018.mp3'},
            { id: 's19', title: 'Surah 19: Maryam', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/019.mp3'},
            { id: 's20', title: 'Surah 20: Ta-Ha', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/020.mp3'},
            { id: 's21', title: 'Surah 21: Al-Anbiya', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/021.mp3'},
            { id: 's22', title: 'Surah 22: Al-Hajj', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/022.mp3'},
            { id: 's23', title: 'Surah 23: Al-Mumenoon', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/023.mp3'},
            { id: 's24', title: 'Surah 24: An-Noor', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/024.mp3'},
            { id: 's25', title: 'Surah 25: Al-Furqan', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/025.mp3'},
            { id: 's26', title: 'Surah 26: Ash-Shu-ara', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/026.mp3'},
            { id: 's27', title: 'Surah 27: An-Naml', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/027.mp3'},
            { id: 's28', title: 'Surah 28: Al-Qasas', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/028.mp3'},
            { id: 's29', title: 'Surah 29: Al-Ankaboot', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/029.mp3'},
            { id: 's30', title: 'Surah 30: Ar-Room', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/030.mp3'},
            { id: 's31', title: 'Surah 31: Luqman', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/031.mp3'},
            { id: 's32', title: 'Surah 32: As-Sajda', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/032.mp3'},
            { id: 's33', title: 'Surah 33: Al-Ahzab', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/033.mp3'},
            { id: 's34', title: 'Surah 34: Saba', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/034.mp3'},
            { id: 's35', title: 'Surah 35: Fatir', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/035.mp3'},
            { id: 's36', title: 'Surah 36: Ya-Seen', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/036.mp3'},
            { id: 's37', title: 'Surah 37: As-Saaffat', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/037.mp3'},
            { id: 's38', title: 'Surah 38: Sad', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/038.mp3'},
            { id: 's39', title: 'Surah 39: Az-Zumar', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/039.mp3'},
            { id: 's40', title: 'Surah 40: Al-Ghafir', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/040.mp3'},
            { id: 's41', title: 'Surah 41: Fussilat', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/041.mp3'},
            { id: 's42', title: 'Surah 42: Ash-Shura', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/042.mp3'},
            { id: 's43', title: 'Surah 43: Az-Zukhruf', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/043.mp3'},
            { id: 's44', title: 'Surah 44: Ad-Dukhan', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/044.mp3'},
            { id: 's45', title: 'Surah 45: Al-Jathiya', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/045.mp3'},
            { id: 's46', title: 'Surah 46: Al-Ahqaf', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/046.mp3'},
            { id: 's47', title: 'Surah 47: Muhammad', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/047.mp3'},
            { id: 's48', title: 'Surah 48: Al-Fath', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/048.mp3'},
            { id: 's49', title: 'Surah 49: Al-Hujraat', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/049.mp3'},
            { id: 's50', title: 'Surah 50: Qaf', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/050.mp3'},
            { id: 's51', title: 'Surah 51: Adh-Dhariyat', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/051.mp3'},
            { id: 's52', title: 'Surah 52: At-Tur', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/052.mp3'},
            { id: 's53', title: 'Surah 53: An-Najm', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/053.mp3'},
            { id: 's54', title: 'Surah 54: Al-Qamar', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/054.mp3'},
            { id: 's55', title: 'Surah 55: Ar-Rahman', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/055.mp3'},
            { id: 's56', title: 'Surah 56: Al-Waqia', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/056.mp3'},
            { id: 's57', title: 'Surah 57: Al-Hadid', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/057.mp3'},
            { id: 's58', title: 'Surah 58: Al-Mujadila', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/058.mp3'},
            { id: 's59', title: 'Surah 59: Al-Hashr', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/059.mp3'},
            { id: 's60', title: 'Surah 60: Al-Mumtahina', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/060.mp3'},
            { id: 's61', title: 'Surah 61: As-Saff', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/061.mp3'},
            { id: 's62', title: 'Surah 62: Al-Jumua', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/062.mp3'},
            { id: 's63', title: 'Surah 63: Al-Munafiqoon', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/063.mp3'},
            { id: 's64', title: 'Surah 64: At-Taghabun', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/064.mp3'},
            { id: 's65', title: 'Surah 65: At-Talaq', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/065.mp3'},
            { id: 's66', title: 'Surah 66: At-Tahrim', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/066.mp3'},
            { id: 's67', title: 'Surah 67: Al-Mulk', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/067.mp3'},
            { id: 's68', title: 'Surah 68: Al-Qalam', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/068.mp3'},
            { id: 's69', title: 'Surah 69: Al-Haaqqa', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/069.mp3'},
            { id: 's70', title: 'Surah 70: Al-Maarij', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/070.mp3'},
            { id: 's71', title: 'Surah 71: Nooh', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/071.mp3'},
            { id: 's72', title: 'Surah 72: Al-Jinn', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/072.mp3'},
            { id: 's73', title: 'Surah 73: Al-Muzzammil', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/073.mp3'},
            { id: 's74', title: 'Surah 74: Al-Muddaththir', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/074.mp3'},
            { id: 's75', title: 'Surah 75: Al-Qiyama', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/075.mp3'},
            { id: 's76', title: 'Surah 76: Al-Insan', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/076.mp3'},
            { id: 's77', title: 'Surah 77: Al-Mursalat', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/077.mp3'},
            { id: 's78', title: 'Surah 78: An-Naba', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/078.mp3'},
            { id: 's79', title: 'Surah 79: An-Naziat', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/079.mp3'},
            { id: 's80', title: 'Surah 80: Abasa', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/080.mp3'},
            { id: 's81', title: 'Surah 81: At-Takwir', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/081.mp3'},
            { id: 's82', title: 'Surah 82: Al-Infitar', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/082.mp3'},
            { id: 's83', title: 'Surah 83: Al-Mutaffifin', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/083.mp3'},
            { id: 's84', title: 'Surah 84: Al-Inshiqaq', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/084.mp3'},
            { id: 's85', title: 'Surah 85: Al-Burooj', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/085.mp3'},
            { id: 's86', title: 'Surah 86: At-Tariq', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/086.mp3'},
            { id: 's87', title: 'Surah 87: Al-Ala', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/087.mp3'},
            { id: 's88', title: 'Surah 88: Al-Ghashiya', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/088.mp3'},
            { id: 's89', title: 'Surah 89: Al-Fajr', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/089.mp3'},
            { id: 's90', title: 'Surah 90: Al-Balad', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/090.mp3'},
            { id: 's91', title: 'Surah 91: Ash-Shams', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/091.mp3'},
            { id: 's92', title: 'Surah 92: Al-Lail', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/092.mp3'},
            { id: 's93', title: 'Surah 93: Ad-Dhuha', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/093.mp3'},
            { id: 's94', title: 'Surah 94: Al-Inshirah', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/094.mp3'},
            { id: 's95', title: 'Surah 95: At-Tin', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/095.mp3'},
            { id: 's96', title: 'Surah 96: Al-Alaq', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/096.mp3'},
            { id: 's97', title: 'Surah 97: Al-Qadr', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/097.mp3'},
            { id: 's98', title: 'Surah 98: Al-Bayyina', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/098.mp3'},
            { id: 's99', title: 'Surah 99: Az-Zalzala', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/099.mp3'},
            { id: 's100', title: 'Surah 100: Al-Adiyat', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/100.mp3'},
            { id: 's101', title: 'Surah 101: Al-Qaria', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/101.mp3'},
            { id: 's102', title: 'Surah 102: At-Takathur', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/102.mp3'},
            { id: 's103', title: 'Surah 103: Al-Asr', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/103.mp3'},
            { id: 's104', title: 'Surah 104: Al-Humaza', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/104.mp3'},
            { id: 's105', title: 'Surah 105: Al-Fil', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/105.mp3'},
            { id: 's106', title: 'Surah 106: Quraish', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/106.mp3'},
            { id: 's107', title: 'Surah 107: Al-Maun', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/107.mp3'},
            { id: 's108', title: 'Surah 108: Al-Kauther', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/108.mp3'},
            { id: 's109', title: 'Surah 109: Al-Kafiroon', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/109.mp3'},
            { id: 's110', title: 'Surah 110: An-Nasr', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/110.mp3'},
            { id: 's111', title: 'Surah 111: Al-Masadd', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/111.mp3'},
            { id: 's112', title: 'Surah 112: Al-Ikhlas', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/112.mp3'},
            { id: 's113', title: 'Surah 113: Al-Falaq', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/113.mp3'},
            { id: 's114', title: 'Surah 114: An-Nas', artist: 'Mishary Al-Afasy and Ibrahim Walk', url: 'https://archive.org/download/AlQuranWithEnglishSaheehIntlTranslation--RecitationByMishariIbnRashidAl-AfasyWithIbrahimWalk/114.mp3'}
        ];

        let currentSongIndex = 0;
        let isPlaying = false;
        let isShuffle = false;
        let isLoop = false;
        let playbackTracklist = [...songs];
        let originalPlaybackTracklist = [...songs]; 
        let favoriteSongIds = [];
        let playlists = []; 
        let currentOpenPlaylistId = null;
        let lastPlaybackState = null;
        let sleepTimerId = null;
        let activeTimerMinutes = 0;
        let playlistIdToDelete = null; // NEW: To track which playlist to delete

        // --- Toast Notification ---
        function showToast(message, duration) {
            duration = duration || 3000;
            toastNotification.textContent = message;
            toastNotification.classList.add('show');
            setTimeout(function() {
                toastNotification.classList.remove('show');
            }, duration);
        }

        // --- Authentication and Data Loading ---
        onAuthStateChanged(auth, function(user) {
            if (user) {
                userId = user.uid;
                console.log("User authenticated with UID:", userId);
                dbUserDocRef = doc(db, "artifacts", appId, "users", userId);
                dbPlaylistsCollectionRef = collection(dbUserDocRef, "playlists");
                
                loadUserDocumentData();
                loadPlaylists();
            } else {
                userId = null;
                console.log("User not authenticated. Player data will not be saved.");
                if (unsubscribeUserDoc) unsubscribeUserDoc();
                if (unsubscribePlaylists) unsubscribePlaylists();
                favoriteSongIds = []; 
                playlists = [];
                renderFavorites();
                renderPlaylists();
                updateActiveTab('library', { isInitialLoad: true });
            }
        });

        async function initAuth() {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Authentication error:", error);
                showToast("Error connecting to services. Data might not save.", 5000);
            }
        }

        // --- UI Logic ---
        function updatePlayerHeader() {
            const songToDisplay = playbackTracklist[currentSongIndex];
            const activeTabButton = document.querySelector('.tab-button.tab-active');
            const activeTab = activeTabButton ? activeTabButton.dataset.tab : 'library';

            if (songToDisplay) {
                songTitleDisplay.textContent = songToDisplay.title;
                songArtistDisplay.textContent = songToDisplay.artist;
                return;
            }
            
            if (activeTab === 'favorites') {
                songTitleDisplay.textContent = "No Surahs in Favorites";
            } else if (activeTab === 'playlists' && !currentOpenPlaylistId) {
                songTitleDisplay.textContent = "No Surahs in Playlist";
            } else if (activeTab === 'playlists' && currentOpenPlaylistId) {
                songTitleDisplay.textContent = "This Playlist is Empty";
            } else {
                songTitleDisplay.textContent = "No Surah Selected";
            }
            songArtistDisplay.textContent = "---";
        }

        function updateNowPlayingIndicator() {
            document.querySelectorAll('.now-playing-indicator').forEach(function(indicator) {
                indicator.style.display = 'none';
            });
            headerNowPlayingIndicator.style.display = 'none';

            if (isPlaying) {
                const currentSong = playbackTracklist[currentSongIndex];
                if (currentSong) {
                    const songItems = document.querySelectorAll(`.song-item[data-song-id="${currentSong.id}"]`);
                    songItems.forEach(function(songItem){
                       const indicator = songItem.querySelector('.now-playing-indicator');
                        if (indicator) {
                            indicator.style.display = 'flex';
                        }
                    });
                    headerNowPlayingIndicator.style.display = 'flex';
                }
            }
        }


        // --- Audio Controls ---
        function loadSong(index, options) {
            options = options || {};
            if (index >= 0 && index < playbackTracklist.length) {
                const song = playbackTracklist[index];
                currentSongIndex = index; 
                audioPlayer.src = song.url;

                audioPlayer.oncanplaythrough = function() {
                    if (options.seekTime) {
                        audioPlayer.currentTime = options.seekTime;
                    }
                    if (isPlaying) {
                        audioPlayer.play().catch(function(e) { console.error("Error playing loaded Surah:", e); });
                    }
                    audioPlayer.oncanplaythrough = null;
                };

            } else {
                currentSongIndex = -1; 
                audioPlayer.src = ''; 
                pauseSong();
            }
            updatePlayerHeader(); 
            updateSelectedSongUI();
            updateNowPlayingIndicator();
        }

        function playSong() {
            if (playbackTracklist.length === 0) {
                 showToast("No song to play.", 3000);
                 return;
            }
            if (currentSongIndex === -1) {
                currentSongIndex = 0;
            }
            
            const songToPlay = playbackTracklist[currentSongIndex];
            if (!audioPlayer.src || audioPlayer.currentSrc !== songToPlay.url) {
                loadSong(currentSongIndex);
            }
            
            if (audioPlayer.src) {
                audioPlayer.play()
                    .then(function() {
                        isPlaying = true;
                        playPauseBtn.innerHTML = '<i class="fas fa-pause fa-2x"></i>';
                        updateNowPlayingIndicator();
                    })
                    .catch(function(error) {
                        console.error("Error playing audio:", error);
                        isPlaying = false;
                        playPauseBtn.innerHTML = '<i class="fas fa-play fa-2x"></i>';
                    });
            }
        }

        function pauseSong() {
            audioPlayer.pause();
            isPlaying = false;
            playPauseBtn.innerHTML = '<i class="fas fa-play fa-2x"></i>';
            updateNowPlayingIndicator();
            savePlaybackState(); 
        }

        playPauseBtn.addEventListener('click', function() {
            if (isPlaying) {
                pauseSong();
            } else {
                playSong();
            }
        });

        nextBtn.addEventListener('click', function() {
            if (playbackTracklist.length === 0) return;
            let nextIndex = currentSongIndex + 1;
            if (nextIndex >= playbackTracklist.length) {
                nextIndex = 0; 
            }
            loadSong(nextIndex);
        });

        prevBtn.addEventListener('click', function() {
            if (playbackTracklist.length === 0) return;
            let prevIndex = currentSongIndex - 1;
            if (prevIndex < 0) {
                prevIndex = playbackTracklist.length - 1;
            }
            loadSong(prevIndex);
        });

        loopBtn.addEventListener('click', function() {
            isLoop = !isLoop;
            audioPlayer.loop = isLoop; 
            loopBtn.classList.toggle('text-[#84cc16]', isLoop);
            loopBtn.classList.toggle('text-gray-400', !isLoop);
            showToast(isLoop ? "Loop current track enabled" : "Loop current track disabled");
        });

        shuffleBtn.addEventListener('click', function() {
            isShuffle = !isShuffle;
            shuffleBtn.classList.toggle('text-[#84cc16]', isShuffle);
            shuffleBtn.classList.toggle('text-gray-400', !isShuffle);
            
            const playingSongId = playbackTracklist[currentSongIndex]?.id;
            
            if (isShuffle) {
                originalPlaybackTracklist = [...playbackTracklist];
                playbackTracklist.sort(function() { return Math.random() - 0.5; });
            } else {
                playbackTracklist = [...originalPlaybackTracklist];
            }

            const newIndex = playingSongId ? playbackTracklist.findIndex(function(s) { return s.id === playingSongId; }) : -1;
            currentSongIndex = newIndex !== -1 ? newIndex : 0;
            
            updateSelectedSongUI();
            updateNowPlayingIndicator();
            showToast(isShuffle ? "Shuffle enabled" : "Shuffle disabled");
        });
        

        audioPlayer.addEventListener('timeupdate', function() {
            if (audioPlayer.duration) {
                const progress = (audioPlayer.currentTime / audioPlayer.duration) * 100;
                progressBar.value = progress;
                currentTimeDisplay.textContent = formatTime(audioPlayer.currentTime);
            }
        });

        audioPlayer.addEventListener('loadedmetadata', function() {
            durationDisplay.textContent = formatTime(audioPlayer.duration);
            progressBar.value = 0; 
        });
        
        audioPlayer.addEventListener('ended', function() {
            if (!audioPlayer.loop) {
                nextBtn.click();
            }
        });


        progressBar.addEventListener('input', function(e) {
            if (audioPlayer.duration) {
                const seekTime = (e.target.value / 100) * audioPlayer.duration;
                audioPlayer.currentTime = seekTime;
            }
        });

        volumeCtrl.addEventListener('input', function(e) {
            audioPlayer.volume = e.target.value;
        });

        function formatTime(seconds) {
            const minutes = Math.floor(seconds / 60);
            const secs = Math.floor(seconds % 60);
            return minutes + ':' + (secs < 10 ? '0' : '') + secs;
        }

        // --- Song List Rendering ---
        function renderSongList(container, tracklistToRender, isPlaylistContext, playlistIdForContext) {
            container.innerHTML = ''; 
            if (!tracklistToRender || tracklistToRender.length === 0) {
                 if (container === libraryView) container.innerHTML = '<p class="text-gray-500 text-center p-4">The Audio Library is empty.</p>';
                return;
            }

            tracklistToRender.forEach(function(song) {
                const songDiv = document.createElement('div');
                songDiv.className = 'song-item p-3 flex justify-between items-center cursor-pointer hover:bg-gray-700 transition-colors duration-150';
                
                const currentSong = playbackTracklist[currentSongIndex];
                if (currentSong && song.id === currentSong.id) {
                     songDiv.classList.add('selected');
                }
                songDiv.dataset.songId = song.id;

                songDiv.innerHTML = `
                    <div class="flex-grow flex items-center" data-action="play">
                        <div class="now-playing-indicator mr-2" style="display: none;">
                           <span></span><span></span><span></span><span></span>
                        </div>
                        <div>
                            <p class="font-semibold truncate">${song.title}</p>
                            <p class="text-sm text-gray-400 truncate">${song.artist}</p>
                        </div>
                    </div>
                    <div class="flex items-center space-x-1">
                        <button data-action="toggleFavorite" data-song-id="${song.id}" class="text-gray-400 hover:text-pink-500 p-2 rounded-full focus:outline-none">
                            <i class="fas fa-heart ${favoriteSongIds.includes(song.id) ? 'text-pink-500' : ''}"></i>
                        </button>
                        ${isPlaylistContext ? 
                            `<button data-action="removeFromPlaylist" data-song-id="${song.id}" data-playlist-id="${playlistIdForContext}" class="text-gray-400 hover:text-red-500 p-2 rounded-full focus:outline-none">
                                <i class="fas fa-trash-alt"></i>
                            </button>` :
                            `<button data-action="addToPlaylist" data-song-id="${song.id}" class="text-gray-400 hover:text-[#84cc16] p-2 rounded-full focus:outline-none">
                                <i class="fas fa-plus-circle"></i>
                            </button>`
                        }
                    </div>
                `;
                
                songDiv.addEventListener('click', function(e) {
                    const actionTarget = e.target.closest('button') || e.target.closest('[data-action="play"]');
                    if (!actionTarget) return;

                    const action = actionTarget.dataset.action;
                    const clickedSongId = songDiv.dataset.songId;

                    if (action === 'play') {
                        playbackTracklist = [...tracklistToRender];
                        originalPlaybackTracklist = [...tracklistToRender];
                        
                        const songIndexToPlay = playbackTracklist.findIndex(function(s) { return s.id === clickedSongId; });

                        if (songIndexToPlay !== -1) {
                            loadSong(songIndexToPlay);
                            playSong();
                        }
                    } else if (action === 'toggleFavorite') {
                        const icon = actionTarget.querySelector('i');
                        toggleFavorite(clickedSongId, icon);
                    } else if (action === 'addToPlaylist') {
                        openAddToPlaylistModal(clickedSongId);
                    } else if (action === 'removeFromPlaylist') {
                        const playlistId = actionTarget.dataset.playlistId;
                        removeSongFromPlaylist(clickedSongId, playlistId);
                    }
                });
                container.appendChild(songDiv);
            });
        }
        
        function updateSelectedSongUI() {
            document.querySelectorAll('.song-item').forEach(function(item) {
                item.classList.remove('selected', 'border-l-4', 'border-[#84cc16]');
                const songId = item.dataset.songId;
                
                const currentSong = playbackTracklist[currentSongIndex];
                if (currentSong && songId === currentSong.id) {
                    item.classList.add('selected');
                }
            });
        }

        // --- Tab Navigation ---
        const tabButtons = document.querySelectorAll('.tab-button');
        const tabContents = document.querySelectorAll('.tab-content');

        tabButtons.forEach(function(button) {
            button.addEventListener('click', function() {
                const tabName = button.dataset.tab;
                updateActiveTab(tabName);
            });
        });
        
        function updateActiveTab(tabName, options) {
            options = options || {};
            
            const currentActiveContent = document.querySelector('.tab-content:not(.hidden)');
            if (currentActiveContent) {
                currentActiveContent.classList.add('fade-out');
            }

            setTimeout(function() {
                if(currentActiveContent) {
                    currentActiveContent.classList.add('hidden');
                    currentActiveContent.classList.remove('fade-out');
                }

                tabButtons.forEach(function(btn) {
                    btn.classList.toggle('tab-active', btn.dataset.tab === tabName);
                    btn.classList.toggle('text-gray-400', btn.dataset.tab !== tabName);
                });
                tabContents.forEach(function(content) {
                    const contentId = content.id.toLowerCase();
                    if (contentId.includes(tabName)) {
                        content.classList.remove('hidden');
                        content.classList.add('fade-in');
                    } else {
                        content.classList.remove('fade-in');
                    }
                });

                currentOpenPlaylistId = null; 
                
                if (tabName === 'library') {
                    renderSongList(libraryView, songs); 
                } else if (tabName === 'favorites') {
                    renderFavorites();
                } else if (tabName === 'playlists') {
                    singlePlaylistSongsView.classList.add('hidden'); 
                    renderPlaylists();
                }
                
                if (options.isInitialLoad) {
                    const songIndex = lastPlaybackState ? songs.findIndex(function(s) { return s.id === lastPlaybackState.songId; }) : 0;
                    const songToLoadIndex = songIndex !== -1 ? songIndex : 0;
                    playbackTracklist = [...songs];
                    originalPlaybackTracklist = [...songs];
                    loadSong(songToLoadIndex, { seekTime: lastPlaybackState ? lastPlaybackState.currentTime : 0 });
                }
                updatePlayerHeader();
                updateNowPlayingIndicator();
                updateSelectedSongUI();
            }, 300); // Duration of the fadeOut animation
        }

        // --- Favorites Management ---
        async function loadUserDocumentData() {
            if (!userId || !dbUserDocRef) {
                 favoriteSongIds = []; renderFavorites(); return;
            }
            if (unsubscribeUserDoc) unsubscribeUserDoc(); 

            unsubscribeUserDoc = onSnapshot(dbUserDocRef, function(docSnap) {
                let initialLoad = lastPlaybackState === null;
                if (docSnap.exists()) {
                    const data = docSnap.data();
                    favoriteSongIds = data.favoriteSongIds || [];
                    lastPlaybackState = data.lastPlayed || null;
                } else {
                    favoriteSongIds = [];
                    lastPlaybackState = null;
                    console.log("User document does not exist yet for UID:", userId);
                }
                renderFavorites();
                if (initialLoad) {
                    updateActiveTab('library', { isInitialLoad: true });
                }
            }, function(error) {
                console.error("Error loading user document:", error);
                showToast("Could not load user data.", 4000);
            });
        }

        async function toggleFavorite(songId, iconElement) {
            if (!userId || !dbUserDocRef) {
                showToast("Cannot save favorite. Not connected.", 3000); return;
            }
            
            iconElement.classList.toggle('text-pink-500');

            const isCurrentlyFavorite = favoriteSongIds.includes(songId);
            const operation = isCurrentlyFavorite ? arrayRemove(songId) : arrayUnion(songId);
            
            try {
                await setDoc(dbUserDocRef, { favoriteSongIds: operation }, { merge: true });
                showToast(isCurrentlyFavorite ? "Removed from Favorites" : "Added to Favorites");
            } catch (error) {
                iconElement.classList.toggle('text-pink-500');
                console.error("Error updating favorites:", error);
                showToast("Error saving favorite.", 3000);
            }
        }

        function renderFavorites() {
            const favSongsData = songs.filter(function(song) { return favoriteSongIds.includes(song.id); });
            noFavoritesMessage.classList.toggle('hidden', favSongsData.length > 0);
            renderSongList(favoriteSongsContainer, favSongsData); // <-- MODIFIED

            const activeTabButton = document.querySelector('.tab-button.tab-active');
            if (activeTabButton && activeTabButton.dataset.tab === 'favorites') {
                updatePlayerHeader();
            }
            updateNowPlayingIndicator();
            updateSelectedSongUI();
        }
        
        // --- Playback State Management ---
        async function savePlaybackState() {
            if (!userId || !dbUserDocRef || currentSongIndex < 0) return;
            
            const songToSave = playbackTracklist[currentSongIndex];
            if (!songToSave) return;
            
            const state = {
                songId: songToSave.id,
                currentTime: audioPlayer.currentTime
            };
            
            try {
                await setDoc(dbUserDocRef, { lastPlayed: state }, { merge: true });
            } catch (error) {
                console.error("Error saving playback state:", error);
            }
        }
        
        window.addEventListener('beforeunload', savePlaybackState);
        
        // --- Kodular Communication Bridge ---
        if (window.AppInventor && window.AppInventor.getWebViewString) {
            function checkForAppCommands() {
                const command = window.AppInventor.getWebViewString();
                if (command === "PAUSE_COMMAND") {
                    pauseSong();
                    window.AppInventor.setWebViewString(""); 
                }
            }
            setInterval(checkForAppCommands, 250);
        }

        // --- Sleep Timer ---
        function updateSleepTimerUI() {
            timerOptions.querySelectorAll('button').forEach(function(btn) {
                btn.classList.remove('timer-button-active');
                if (parseInt(btn.dataset.minutes, 10) === activeTimerMinutes) {
                    btn.classList.add('timer-button-active');
                }
            });
            if (activeTimerMinutes > 0) {
                sleepTimerTitle.textContent = "Timer set for " + activeTimerMinutes + " min";
            } else {
                sleepTimerTitle.textContent = "Sleep Timer";
            }
        }

        function setSleepTimer(minutes) {
            if(sleepTimerId) clearTimeout(sleepTimerId);
            activeTimerMinutes = minutes;
            
            if (minutes > 0) {
                sleepTimerId = setTimeout(function() {
                    pauseSong();
                    showToast("Sleep timer finished.");
                    activeTimerMinutes = 0;
                    sleepTimerBtn.classList.remove('text-[#84cc16]');
                    sleepTimerBtn.classList.add('text-gray-400');
                    updateSleepTimerUI();
                }, minutes * 60 * 1000);
                
                sleepTimerBtn.classList.add('text-[#84cc16]');
                sleepTimerBtn.classList.remove('text-gray-400');
                showToast("Sleep timer set for " + minutes + " minutes.");
            } else {
                sleepTimerBtn.classList.remove('text-[#84cc16]');
                sleepTimerBtn.classList.add('text-gray-400');
                showToast("Sleep timer turned off.");
            }
            updateSleepTimerUI();
            sleepTimerModal.classList.remove('active');
        }

        sleepTimerBtn.addEventListener('click', function() {
            updateSleepTimerUI();
            sleepTimerModal.classList.add('active');
        });

        cancelSleepTimerBtn.addEventListener('click', function() {
            sleepTimerModal.classList.remove('active');
        });

        timerOptions.addEventListener('click', function(e) {
            if(e.target.tagName === 'BUTTON') {
                const minutes = parseInt(e.target.dataset.minutes, 10);
                setSleepTimer(minutes);
            }
        });


        // --- Playlist Management ---
        createPlaylistBtn.addEventListener('click', function() {
            newPlaylistNameInput.value = '';
            createPlaylistModal.classList.add('active');
            newPlaylistNameInput.focus();
        });
        cancelCreatePlaylistBtn.addEventListener('click', function() { createPlaylistModal.classList.remove('active'); });

        savePlaylistBtn.addEventListener('click', async function() {
            if (!userId || !dbPlaylistsCollectionRef) {
                 showToast("Cannot save playlist. Not connected.", 3000); return;
            }
            const playlistName = newPlaylistNameInput.value.trim();
            if (!playlistName) {
                showToast("Playlist name cannot be empty.", 3000); return;
            }
            if (playlists.find(function(p) { return p.name.toLowerCase() === playlistName.toLowerCase(); })) {
                showToast("A playlist with this name already exists.", 3000); return;
            }
            try {
                await addDoc(dbPlaylistsCollectionRef, {
                    name: playlistName,
                    songIds: [],
                    createdAt: new Date()
                });
                createPlaylistModal.classList.remove('active');
                showToast(`Playlist "${playlistName}" created.`);
            } catch (error) {
                console.error("Error creating playlist:", error);
                showToast("Error creating playlist.", 3000);
            }
        });

        async function loadPlaylists() {
            if (!userId || !dbPlaylistsCollectionRef) {
                 playlists = []; renderPlaylists(); return;
            }
            if (unsubscribePlaylists) unsubscribePlaylists();

            const q = query(dbPlaylistsCollectionRef);
            unsubscribePlaylists = onSnapshot(q, function(querySnapshot) {
                playlists = [];
                querySnapshot.forEach(function(doc) {
                    playlists.push({ id: doc.id, ...doc.data() });
                });
                playlists.sort(function(a,b) { return a.name.localeCompare(b.name); });

                renderPlaylists();
                
                if (currentOpenPlaylistId) {
                    const stillExists = playlists.find(function(p) { return p.id === currentOpenPlaylistId; });
                    if (stillExists) {
                        renderSongsInPlaylist(currentOpenPlaylistId);
                    } else { 
                        showPlaylistsList(); 
                    }
                }
            }, function(error) {
                console.error("Error loading playlists:", error);
            });
        }
        
        function renderPlaylists() {
            myPlaylistsContainer.innerHTML = '';
            noPlaylistsMessage.classList.toggle('hidden', playlists.length > 0);

            if (playlists.length > 0) {
                playlists.forEach(function(playlist) {
                    const div = document.createElement('div');
                    div.className = 'p-3 bg-gray-800 rounded-lg mb-2 flex justify-between items-center cursor-pointer hover:bg-gray-700 transition-colors duration-150';
                    div.innerHTML = `
                        <div class="font-semibold truncate flex-grow" data-action="open-playlist">
                           ${playlist.name} (${(playlist.songIds && playlist.songIds.length) || 0} songs)
                        </div>
                        <div class="flex-shrink-0">
                            <button data-playlist-id="${playlist.id}" data-playlist-name="${playlist.name}" data-action="delete-playlist" class="text-red-500 hover:text-red-400 p-2 rounded-full focus:outline-none"><i class="fas fa-trash-alt"></i></button>
                        </div>
                    `;

                    div.addEventListener('click', function(e) {
                        const target = e.target.closest('[data-action]');
                        if (!target) return;
                        
                        const action = target.dataset.action;
                        
                        if (action === 'delete-playlist') {
                            const playlistId = target.dataset.playlistId;
                            const playlistName = target.dataset.playlistName;
                            // UPDATED: Open custom modal instead of confirm()
                            openDeleteConfirmationModal(playlistId, playlistName);
                        } else if (action === 'open-playlist') {
                            showSongsForPlaylist(playlist.id);
                        }
                    });
                    myPlaylistsContainer.appendChild(div);
                });
            }
            
            const activeTabButton = document.querySelector('.tab-button.tab-active');
            if (activeTabButton && activeTabButton.dataset.tab === 'playlists' && !currentOpenPlaylistId) {
                 updatePlayerHeader();
            }
        }
        
        // NEW: Functions to handle the delete confirmation modal
        function openDeleteConfirmationModal(playlistId, playlistName) {
            playlistIdToDelete = playlistId;
            deleteModalText.textContent = `Are you sure you want to delete the playlist "${playlistName}"? This cannot be undone.`;
            deletePlaylistModal.classList.add('active');
        }

        function closeDeleteConfirmationModal() {
            deletePlaylistModal.classList.remove('active');
            playlistIdToDelete = null;
        }

        async function deletePlaylist(playlistId) {
             if (!userId || !dbPlaylistsCollectionRef) {
                 showToast("Cannot delete playlist. Not connected.", 3000); return;
            }
            try {
                await deleteDoc(doc(dbPlaylistsCollectionRef, playlistId));
                showToast("Playlist deleted.");
            } catch (error) {
                console.error("Error deleting playlist:", error);
                showToast("Error deleting playlist.", 3000);
            }
        }

        function openAddToPlaylistModal(songId) {
            if (!userId || !dbPlaylistsCollectionRef) { showToast("Connect to save changes.", 3000); return; }
            songIdToAddToPlaylistInput.value = songId;
            playlistSelectionContainer.innerHTML = '';
            noPlaylistsForAddingMessage.classList.toggle('hidden', playlists.length > 0);

            if (playlists.length > 0) {
                playlists.forEach(function(playlist) {
                    const isSongInPlaylist = playlist.songIds && playlist.songIds.includes(songId);

                    const pDiv = document.createElement('div');
                    pDiv.className = 'p-2 border-b border-gray-700 flex justify-between items-center last:border-b-0';
                    pDiv.innerHTML = `
                        <span class="truncate">${playlist.name}</span>
                        <button data-playlist-id="${playlist.id}" ${isSongInPlaylist ? 'disabled class="text-gray-500 cursor-not-allowed p-1 rounded"' : 'class="text-[#84cc16] hover:text-[#65a30d] p-1 rounded"'}">
                            ${isSongInPlaylist ? '<i class="fas fa-check-circle mr-1"></i> Added' : '<i class="fas fa-plus-circle mr-1"></i> Add'}
                        </button>
                    `;
                    if (!isSongInPlaylist) {
                        pDiv.querySelector('button').addEventListener('click', async function() {
                            await addSongToPlaylist(songId, playlist.id);
                            addToPlaylistModal.classList.remove('active'); 
                        });
                    }
                    playlistSelectionContainer.appendChild(pDiv);
                });
            }
            addToPlaylistModal.classList.add('active');
        }
        cancelAddToPlaylistBtn.addEventListener('click', function() { addToPlaylistModal.classList.remove('active'); });

        async function addSongToPlaylist(songId, playlistId) {
            if (!userId || !dbPlaylistsCollectionRef) {
                 showToast("Cannot add to playlist. Not connected.", 3000); return;
            }
            const playlistRef = doc(dbPlaylistsCollectionRef, playlistId);
            try {
                await updateDoc(playlistRef, { songIds: arrayUnion(songId) });
                const playlist = playlists.find(function(p) { return p.id === playlistId; });
                showToast(`Added to playlist "${playlist ? playlist.name : 'playlist'}".`);
            } catch (error) {
                console.error("Error adding song to playlist:", error);
                showToast("Error adding Surah to playlist.", 3000);
            }
        }

        async function removeSongFromPlaylist(songId, playlistId) {
             if (!userId || !dbPlaylistsCollectionRef) {
                 showToast("Cannot remove from playlist. Not connected.", 3000); return;
            }
            const playlistRef = doc(dbPlaylistsCollectionRef, playlistId);
            try {
                await updateDoc(playlistRef, { songIds: arrayRemove(songId) });
                const playlist = playlists.find(function(p) { return p.id === playlistId; });
                showToast(`Removed from playlist "${playlist ? playlist.name : 'playlist'}".`);
            } catch (error) {
                console.error("Error removing song from playlist:", error);
                showToast("Error removing Surah from playlist.", 3000);
            }
        }

        function showSongsForPlaylist(playlistId) {
            const playlist = playlists.find(function(p) { return p.id === playlistId; });
            if (!playlist) {
                console.error("Playlist not found:", playlistId);
                showPlaylistsList(); 
                return;
            }
            currentOpenPlaylistId = playlistId;

            document.getElementById('playlistsView').classList.add('hidden');
            singlePlaylistSongsView.classList.remove('hidden');
            currentPlaylistNameHeader.textContent = playlist.name;
            renderSongsInPlaylist(playlistId);
        }
        
        function renderSongsInPlaylist(playlistId) {
            const playlist = playlists.find(function(p) { return p.id === playlistId; });
            songsInPlaylistContainer.innerHTML = ''; 
            
            const playlistSongsData = (playlist && playlist.songIds) ? songs.filter(function(song) { return playlist.songIds.includes(song.id); }) : [];
            
            noSongsInPlaylistMessage.classList.toggle('hidden', playlistSongsData.length > 0);
            renderSongList(songsInPlaylistContainer, playlistSongsData, true, playlistId);
            
            playbackTracklist = isShuffle ? [...playlistSongsData].sort(function() { return Math.random() - 0.5; }) : [...playlistSongsData];
            originalPlaybackTracklist = [...playlistSongsData];
            
            const currentSong = playbackTracklist[currentSongIndex];
            const stillPlayingValidSong = currentSong && playbackTracklist.some(function(s) { return s.id === currentSong.id; });

            if (!stillPlayingValidSong) {
                loadSong(0);
            }
            
            updatePlayerHeader({view: 'singlePlaylist', isEmpty: playlistSongsData.length === 0});
            updateSelectedSongUI();
        }


        backToPlaylistsBtn.addEventListener('click', showPlaylistsList);

        function showPlaylistsList() {
            updateActiveTab('playlists');
        }


        // --- Initial Setup ---
        document.addEventListener('DOMContentLoaded', function() {
            initAuth(); 
            
            // NEW: Event listeners for the delete confirmation modal
            confirmDeletePlaylistBtn.addEventListener('click', () => {
                if (playlistIdToDelete) {
                    deletePlaylist(playlistIdToDelete);
                }
                closeDeleteConfirmationModal();
            });

            cancelDeletePlaylistBtn.addEventListener('click', closeDeleteConfirmationModal);
        });

    </script>
</body>
</html>
