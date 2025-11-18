# rohan02468.github.io
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TaskFlow - Digital Design App</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Load Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <!-- Custom Tailwind Configuration for TaskFlow's clean aesthetic -->
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        'primary-teal': '#20C997', // Your chosen action color
                        'accent-blue': '#4A90E2',  // School
                        'accent-orange': '#F5A623', // Work
                        'accent-green': '#7ED321', // Personal
                        'neutral-light': '#F8F9FA',
                    },
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <style>
        /* Ensures the app container is centered and takes full mobile height */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f1f5f9;
        }
        #app-container {
            max-width: 450px;
            margin: 0 auto;
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
        }
        /* Custom scrollbar for better mobile feel */
        .scrollable-content::-webkit-scrollbar {
            display: none;
        }
        .scrollable-content {
            -ms-overflow-style: none;  /* IE and Edge */
            scrollbar-width: none;  /* Firefox */
        }
    </style>
</head>
<body class="bg-neutral-light">

    <!-- TaskFlow App Container -->
    <div id="app-container" class="bg-white rounded-lg overflow-hidden">
        
        <!-- Header -->
        <header class="flex justify-between items-center p-4 border-b border-gray-100 bg-white sticky top-0 z-10 shadow-sm">
            <h1 class="text-xl font-bold text-gray-800">TaskFlow</h1>
            <div class="flex space-x-3">
                <button onclick="changeView('premium')" class="text-gray-500 hover:text-primary-teal transition duration-150">
                    <i data-lucide="crown" class="w-5 h-5"></i>
                </button>
                <button onclick="changeView('settings')" class="text-gray-500 hover:text-primary-teal transition duration-150">
                    <i data-lucide="settings" class="w-5 h-5"></i>
                </button>
            </div>
        </header>

        <!-- Main Content View Area -->
        <main id="content-view" class="flex-grow scrollable-content overflow-y-auto">
            <!-- Content will be injected here (Dashboard, Create, etc.) -->
        </main>

        <!-- Floating Add Button -->
        <button id="add-task-btn" onclick="changeView('create')" class="fixed bottom-20 right-4 md:right-[calc(50vw-200px)] z-20 w-14 h-14 bg-primary-teal text-white rounded-full shadow-xl flex items-center justify-center transition duration-200 hover:scale-105 active:scale-95">
            <i data-lucide="plus" class="w-7 h-7"></i>
        </button>

        <!-- Navigation Bar (Footer) -->
        <footer class="flex justify-around items-center h-16 border-t border-gray-100 bg-white sticky bottom-0 z-10">
            <button id="nav-dashboard" onclick="changeView('dashboard')" class="nav-item flex flex-col items-center p-2 text-primary-teal">
                <i data-lucide="layout-dashboard" class="w-5 h-5"></i>
                <span class="text-xs mt-1">Dashboard</span>
            </button>
            <button id="nav-calendar" onclick="changeView('calendar')" class="nav-item flex flex-col items-center p-2 text-gray-400">
                <i data-lucide="calendar" class="w-5 h-5"></i>
                <span class="text-xs mt-1">Calendar</span>
            </button>
            <button id="nav-collaboration" onclick="changeView('collaboration')" class="nav-item flex flex-col items-center p-2 text-gray-400">
                <i data-lucide="users" class="w-5 h-5"></i>
                <span class="text-xs mt-1">Share</span>
            </button>
            <button id="nav-tasks" onclick="changeView('tasks')" class="nav-item flex flex-col items-center p-2 text-gray-400">
                <i data-lucide="list-checks" class="w-5 h-5"></i>
                <span class="text-xs mt-1">Tasks</span>
            </button>
        </footer>

    </div>

    <!-- Firebase SDK Imports -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, collection, query, where, onSnapshot, updateDoc, Timestamp, addDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";


        // --- GLOBAL APP STATE & FIREBASE INIT ---

        let app, db, auth, userId = null;
        let currentView = 'dashboard';
        const APP_ID = typeof __app_id !== 'undefined' ? __app_id : 'default-taskflow-app';
        let isAuthReady = false;
        
        const FIREBASE_CONFIG = (typeof __firebase_config !== 'undefined' && __firebase_config)
            ? JSON.parse(__firebase_config)
            : {};
        
        // Log setup for debugging
        setLogLevel('Debug');


        // --- DATA STRUCTURES & CONFIG ---

        const WORKSPACES = {
            'School': 'accent-blue',
            'Work': 'accent-orange',
            'Personal': 'accent-green',
            'Shared': 'gray-400',
        };

        const getCollectionRef = (dbInstance, uid) => {
            const path = `/artifacts/${APP_ID}/users/${uid}/tasks`;
            return collection(dbInstance, path);
        };

        // --- AUTHENTICATION & INITIALIZATION ---

        const initFirebase = async () => {
            try {
                app = initializeApp(FIREBASE_CONFIG);
                db = getFirestore(app);
                auth = getAuth(app);

                const token = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
                
                if (token) {
                    await signInWithCustomToken(auth, token);
                } else {
                    await signInAnonymously(auth);
                }

                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        userId = user.uid;
                        isAuthReady = true;
                        console.log('User signed in. UID:', userId);
                        // Once authenticated, load the initial view with data
                        changeView(currentView);
                    } else {
                        console.error('Authentication failed.');
                    }
                });

            } catch (error) {
                console.error("Firebase Initialization Error:", error);
            }
        };


        // --- CRUD OPERATIONS ---

        const createNewTask = async (taskName, dueDate, workspace) => {
            if (!isAuthReady || !taskName) return console.error('Auth not ready or missing task name.');

            try {
                const tasksRef = getCollectionRef(db, userId);
                await addDoc(tasksRef, {
                    name: taskName,
                    dueDate: dueDate ? Timestamp.fromDate(new Date(dueDate)) : null,
                    workspace: workspace,
                    isComplete: false,
                    createdAt: Timestamp.now(),
                    userId: userId 
                });
                console.log('Task created successfully');
                // Auto switch back to dashboard after successful creation
                changeView('dashboard'); 
            } catch (e) {
                console.error("Error adding document: ", e);
            }
        };

        const toggleTaskCompletion = async (taskId, currentStatus) => {
            if (!isAuthReady || !taskId) return;

            try {
                const taskDocRef = doc(getCollectionRef(db, userId), taskId);
                await updateDoc(taskDocRef, {
                    isComplete: !currentStatus
                });
                console.log('Task status updated:', !currentStatus);
            } catch (e) {
                console.error("Error updating document: ", e);
            }
        };


        // --- RENDERING FUNCTIONS ---

        const renderDashboard = () => {
            const content = document.getElementById('content-view');
            content.innerHTML = `
                <div class="p-5">
                    <h2 class="text-3xl font-extrabold text-gray-900 mb-1">Good Afternoon!</h2>
                    <p class="text-gray-500 mb-6">Your tasks keep you on TaskFlow.</p>
                    
                    <h3 class="text-xl font-semibold text-gray-700 mb-3">⚡️ Quick View: Upcoming</h3>
                    <div id="tasks-list" class="space-y-3 mb-8">
                        <!-- Tasks will be injected here -->
                        <div class="text-center text-gray-500 p-8 border border-neutral-light rounded-xl">Loading tasks...</div>
                    </div>

                    <h3 class="text-xl font-semibold text-gray-700 mb-3">📚 My Workspaces</h3>
                    <div class="grid grid-cols-2 gap-3">
                        ${Object.keys(WORKSPACES).map(ws => `
                            <button class="flex items-center p-4 bg-${WORKSPACES[ws]} bg-opacity-10 rounded-xl hover:shadow-md transition duration-150">
                                <span class="w-3 h-3 rounded-full bg-${WORKSPACES[ws]} mr-3"></span>
                                <span class="font-medium text-gray-800">${ws}</span>
                            </button>
                        `).join('')}
                    </div>
                </div>
            `;
            if (isAuthReady) {
                listenForTasks('dashboard');
            } else {
                content.querySelector('#tasks-list').innerHTML = `<div class="text-center text-gray-500 p-8 border border-neutral-light rounded-xl">Authenticating...</div>`;
            }
        };

        const renderCreateTask = () => {
            const content = document.getElementById('content-view');
            content.innerHTML = `
                <div class="p-5">
                    <h2 class="text-2xl font-bold text-gray-900 mb-6">Create New Task</h2>
                    <form id="task-form" class="space-y-5">
                        
                        <div>
                            <label for="taskName" class="block text-lg font-medium text-gray-700 mb-1">What needs to get done?</label>
                            <input type="text" id="taskName" required placeholder="e.g., Finish MYP Report" class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:border-primary-teal focus:ring-1 focus:ring-primary-teal text-gray-800">
                        </div>

                        <div>
                            <label for="dueDate" class="block text-lg font-medium text-gray-700 mb-1">Due Date</label>
                            <input type="datetime-local" id="dueDate" class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:border-primary-teal focus:ring-1 focus:ring-primary-teal text-gray-800">
                        </div>

                        <div>
                            <label class="block text-lg font-medium text-gray-700 mb-2">Add to Workspace</label>
                            <div id="workspace-selector" class="flex flex-wrap gap-3">
                                ${Object.keys(WORKSPACES).map(ws => `
                                    <input type="radio" id="ws-${ws}" name="workspace" value="${ws}" class="hidden" ${ws === 'School' ? 'checked' : ''}>
                                    <label for="ws-${ws}" class="flex items-center px-4 py-2 border-2 border-gray-200 rounded-full cursor-pointer transition-all duration-200 hover:shadow-sm">
                                        <span class="w-3 h-3 rounded-full mr-2 bg-${WORKSPACES[ws]}"></span>
                                        <span class="text-sm font-medium text-gray-700">${ws}</span>
                                    </label>
                                `).join('')}
                            </div>
                        </div>

                        <div class="border-t pt-5 mt-5 border-gray-100">
                             <button type="submit" class="w-full py-3 bg-primary-teal text-white font-semibold rounded-xl shadow-lg hover:bg-opacity-90 transition duration-150">Save Task</button>
                        </div>

                    </form>
                </div>
            `;
            // Attach form submission listener
            document.getElementById('task-form').addEventListener('submit', (e) => {
                e.preventDefault();
                const taskName = document.getElementById('taskName').value;
                const dueDate = document.getElementById('dueDate').value;
                const workspace = document.querySelector('input[name="workspace"]:checked').value;
                createNewTask(taskName, dueDate, workspace);
            });
        };
        
        const renderCollaboration = () => {
            document.getElementById('content-view').innerHTML = `
                <div class="p-5">
                    <h2 class="text-2xl font-bold text-gray-900 mb-6">TaskFlow Share</h2>
                    <p class="text-sm text-gray-500 mb-6">Collaboration is a <span class="font-bold text-primary-teal">Premium Feature</span>. Here is the mock-up based on your design brief.</p>
                    
                    <h3 class="text-xl font-semibold text-gray-700 mb-3">Current Shared Projects</h3>
                    <div class="space-y-3 mb-8">
                        <div class="p-4 border border-gray-200 rounded-xl bg-gray-50">
                            <p class="font-medium text-gray-800">Family Groceries</p>
                            <p class="text-xs text-gray-500">Shared with: Mom, Dad, Sibling (3 people)</p>
                        </div>
                        <div class="p-4 border border-gray-200 rounded-xl bg-gray-50">
                            <p class="font-medium text-gray-800">Q3 Project Team</p>
                            <p class="text-xs text-gray-500">Shared with: John, Sarah (2 people)</p>
                        </div>
                    </div>
                    
                    <button class="w-full py-3 bg-gray-200 text-gray-700 font-semibold rounded-xl transition duration-150 hover:bg-gray-300">
                        Create New Shared Space
                    </button>
                    
                    <div class="mt-8 p-4 bg-primary-teal/10 border-l-4 border-primary-teal rounded-lg">
                        <p class="font-bold text-primary-teal">Upgrade to Premium</p>
                        <p class="text-sm text-gray-700">Unlock real-time team management and synced integration with other platforms.</p>
                    </div>

                </div>
            `;
        };
        
        const renderPremium = () => {
            document.getElementById('content-view').innerHTML = `
                <div class="p-5 text-center">
                    <i data-lucide="crown" class="w-12 h-12 text-yellow-500 mx-auto mb-4 fill-yellow-500"></i>
                    <h2 class="text-3xl font-extrabold text-gray-900 mb-2">Unleash Your Productivity</h2>
                    <p class="text-gray-500 mb-8">Go Premium to de-stress your life and manage complex projects effortlessly.</p>
                    
                    <div class="grid grid-cols-2 gap-4 mb-10">
                        <div class="p-4 border-2 border-primary-teal bg-primary-teal/10 rounded-xl">
                            <p class="font-bold text-lg text-primary-teal">FREE</p>
                            <ul class="text-sm text-gray-600 mt-2 space-y-1">
                                <li>Unlimited Tasks</li>
                                <li>Set Reminders</li>
                                <li>Color Coding</li>
                            </ul>
                        </div>
                         <div class="p-4 border-2 border-primary-teal bg-primary-teal rounded-xl text-white">
                            <p class="font-bold text-lg">PREMIUM</p>
                            <ul class="text-sm mt-2 space-y-1">
                                <li>Team Collaboration</li>
                                <li>Subtasks & Timeline</li>
                                <li>Platform Synchronization</li>
                            </ul>
                        </div>
                    </div>

                    <p class="text-sm text-gray-500 mb-2">Start your journey today:</p>
                    <button class="w-full py-4 bg-yellow-500 text-white font-bold rounded-xl shadow-lg hover:bg-yellow-600 transition duration-150">
                        Start 7-Day Free Trial
                    </button>
                    <p class="text-xs text-gray-400 mt-2">Then $4.99/month or $49.99/year.</p>

                </div>
            `;
            // Re-render Lucide icons for this view
            lucide.createIcons();
        };

        const renderPlaceholder = (title, message) => {
            document.getElementById('content-view').innerHTML = `
                <div class="p-5 text-center">
                    <h2 class="text-2xl font-bold text-gray-900 mb-4">${title}</h2>
                    <div class="p-10 border-2 border-dashed border-gray-300 rounded-xl bg-gray-50">
                        <i data-lucide="info" class="w-8 h-8 text-gray-500 mx-auto mb-3"></i>
                        <p class="text-gray-600">${message}</p>
                    </div>
                </div>
            `;
            // Re-render Lucide icons for this view
            lucide.createIcons();
        }

        // --- REAL-TIME DATA LISTENER ---

        let unsubscribeTaskListener = null;

        const listenForTasks = (view) => {
            if (!isAuthReady || view !== 'dashboard') return;

            // Stop any existing listener
            if (unsubscribeTaskListener) {
                unsubscribeTaskListener();
            }

            const tasksRef = getCollectionRef(db, userId);
            // Query for all tasks, sorted by due date, showing incomplete first
            const q = query(tasksRef, where('isComplete', '==', false)); // Only show INCOMPLETE tasks on the dashboard Quick View

            unsubscribeTaskListener = onSnapshot(q, (querySnapshot) => {
                const tasksContainer = document.getElementById('tasks-list');
                if (!tasksContainer) return;

                let tasksHtml = '';
                if (querySnapshot.empty) {
                    tasksHtml = `<div class="text-center text-gray-500 p-8 border border-neutral-light rounded-xl">No tasks found! Click + to add one.</div>`;
                } else {
                    querySnapshot.forEach((doc) => {
                        const task = doc.data();
                        const id = doc.id;
                        const date = task.dueDate ? task.dueDate.toDate().toLocaleString('en-US', { month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit' }) : 'No Due Date';
                        const colorClass = WORKSPACES[task.workspace] || 'gray-400';

                        tasksHtml += `
                            <div class="flex items-start p-4 bg-white rounded-xl shadow-sm border border-gray-100">
                                <div class="w-1 h-10 rounded-full bg-${colorClass} mt-1"></div>
                                <div class="flex-grow ml-4">
                                    <p class="font-semibold text-gray-800">${task.name}</p>
                                    <p class="text-xs text-gray-500">${date}</p>
                                </div>
                                <input type="checkbox" data-task-id="${id}" ${task.isComplete ? 'checked' : ''} class="w-5 h-5 mt-1 rounded text-primary-teal border-gray-300 focus:ring-primary-teal" />
                            </div>
                        `;
                    });
                }
                tasksContainer.innerHTML = tasksHtml;

                // Re-attach completion listeners
                tasksContainer.querySelectorAll('input[type="checkbox"]').forEach(checkbox => {
                    checkbox.addEventListener('change', (e) => {
                        const taskId = e.target.dataset.taskId;
                        const isComplete = e.target.checked;
                        toggleTaskCompletion(taskId, !isComplete); // toggleTaskCompletion takes current status, so pass opposite of new status
                    });
                });
            }, (error) => {
                console.error("Firestore Listener Error:", error);
                const tasksContainer = document.getElementById('tasks-list');
                if (tasksContainer) tasksContainer.innerHTML = `<div class="text-center text-red-500 p-8 border border-red-300 rounded-xl">Error loading tasks.</div>`;
            });
        };


        // --- VIEW MANAGEMENT ---

        window.changeView = (newView) => {
            currentView = newView;
            document.querySelectorAll('.nav-item').forEach(btn => {
                const btnView = btn.id.split('-')[1];
                if (btnView === newView) {
                    btn.classList.add('text-primary-teal');
                    btn.classList.remove('text-gray-400');
                } else {
                    btn.classList.add('text-gray-400');
                    btn.classList.remove('text-primary-teal');
                }
            });

            // Handle the Add button visibility
            const addButton = document.getElementById('add-task-btn');
            if (newView === 'dashboard' || newView === 'tasks') {
                addButton.classList.remove('hidden');
            } else {
                addButton.classList.add('hidden');
            }

            switch (newView) {
                case 'dashboard':
                    renderDashboard();
                    break;
                case 'create':
                    renderCreateTask();
                    break;
                case 'collaboration':
                    renderCollaboration();
                    break;
                case 'premium':
                    renderPremium();
                    break;
                case 'calendar':
                    renderPlaceholder('Calendar View', 'A dynamic calendar view would be implemented here, showing deadlines. (Prototype)');
                    break;
                case 'tasks':
                    renderPlaceholder('All Tasks', 'This screen would show all tasks (complete and incomplete) for detailed management. (Prototype)');
                    break;
                default:
                    renderDashboard();
            }
        };

        // --- ENTRY POINT ---
        document.addEventListener('DOMContentLoaded', () => {
            initFirebase();
            // Initial view is set inside initFirebase after auth is ready
            // lucide.createIcons(); // Icons are created after content injection in changeView
        });
    </script>
</body>
</html>
