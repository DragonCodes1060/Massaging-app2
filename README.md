<html lang="en">
<head>
    <html charset="UTF-8">
    <html name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Messaging Web App</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Inter Font -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        /* Custom scrollbar for better aesthetics */
        .overflow-y-auto::-webkit-scrollbar {
            width: 8px;
        }
        .overflow-y-auto::-webkit-scrollbar-track {
            background: #f1f1f1;
            border-radius: 10px;
        }
        .overflow-y-auto::-webkit-scrollbar-thumb {
            background: #888;
            border-radius: 10px;
        }
        .overflow-y-auto::-webkit-scrollbar-thumb:hover {
            background: #555;
        }
        /* Hide scrollbar for editing message content */
        textarea::-webkit-scrollbar {
            width: 0px;
            background: transparent; /* make scrollbar transparent */
        }
        .group:hover .group-hover\:opacity-100 {
            opacity: 1;
        }
    </style>
</head>
<body class="antialiased bg-gray-100 text-gray-900 min-h-screen flex flex-col">

    <!-- Main App Container -->
    <div id="app-root" class="flex-1 flex flex-col">
        <!-- Content will be injected here by JavaScript -->
    </div>

    <!-- Custom Alert/Message Box -->
    <div id="message-box" class="hidden fixed inset-0 bg-gray-800 bg-opacity-75 flex items-center justify-center z-[999]">
        <div class="bg-white p-6 rounded-xl shadow-2xl max-w-sm w-full mx-4 border border-gray-200">
            <p id="message-box-content" class="text-gray-800 text-center text-lg font-medium mb-5"></p>
            <button id="message-box-close" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg focus:outline-none focus:shadow-outline transition duration-300">
                OK
            </button>
        </div>
    </div>


    <script>
        // Global state variables (simulating React's useState)
        let isLoggedIn = false;
        let currentUser = null; // { id: '...', username: '...' }
        let chats = [];
        let messages = [];
        let selectedChat = null; // The currently active chat
        let showRegister = false;
        let showCreateChatModal = false;
        let createChatType = 'one-on-one'; // 'one-on-one' or 'group'
        let editingMessageId = null;
        let editingMessageContent = '';
        let messagesEndRef = null; // Will be set to the div at the end of messages

        // Mock Data - In a real app, this would come from a backend API and WebSocket
        let mockUsers = [
            { id: 'user1', username: 'Alice', password: 'password123' },
            { id: 'user2', username: 'Bob', password: 'password123' },
            { id: 'user3', username: 'Charlie', password: 'password123' },
            { id: 'user4', username: 'David', password: 'password123' },
        ];

        let mockChats = [
            { id: 'chat1', name: 'Alice & Bob', isGroup: false, participants: ['user1', 'user2'] },
            { id: 'chat2', name: 'General Discussion', isGroup: true, participants: ['user1', 'user2', 'user3'] },
            { id: 'chat3', name: 'Alice & Charlie', isGroup: false, participants: ['user1', 'user3'] },
        ];

        let mockMessages = [
            { id: 'msg1', chatId: 'chat1', senderId: 'user1', content: 'Hi Bob!', timestamp: Date.now() - 60000, isEdited: false },
            { id: 'msg2', chatId: 'chat1', senderId: 'user2', content: 'Hey Alice!', timestamp: Date.now() - 30000, isEdited: false },
            { id: 'msg3', chatId: 'chat2', senderId: 'user1', content: 'Hello everyone!', timestamp: Date.now() - 20000, isEdited: false },
            { id: 'msg4', chatId: 'chat2', senderId: 'user3', content: 'Hi Alice!', timestamp: Date.now() - 10000, isEdited: false },
            { id: 'msg5', chatId: 'chat3', senderId: 'user1', content: 'Hey Charlie!', timestamp: Date.now() - 5000, isEdited: false },
        ];

        let messageInterval = null;

        // --- Utility Functions (Simulating React Hooks/Lifecycle) ---

        function renderApp() {
            const appRoot = document.getElementById('app-root');
            appRoot.innerHTML = ''; // Clear existing content

            if (!isLoggedIn) {
                appRoot.appendChild(AuthComponent());
            } else {
                appRoot.appendChild(MainLayout());
            }
        }

        function showMessageBox(message) {
            const msgBox = document.getElementById('message-box');
            const msgContent = document.getElementById('message-box-content');
            const msgCloseBtn = document.getElementById('message-box-close');

            msgContent.textContent = message;
            msgBox.classList.remove('hidden');

            const closeHandler = () => {
                msgBox.classList.add('hidden');
                msgCloseBtn.removeEventListener('click', closeHandler);
            };
            msgCloseBtn.addEventListener('click', closeHandler);
        }

        function scrollToBottom() {
            if (messagesEndRef) {
                messagesEndRef.scrollIntoView({ behavior: "smooth" });
            }
        }

        // Simulate useEffect for message updates and scrolling
        function setupMessageSimulation() {
            if (messageInterval) {
                clearInterval(messageInterval);
            }
            if (currentUser && selectedChat) {
                messageInterval = setInterval(() => {
                    const newMockMessage = {
                        id: `msg${Date.now()}`,
                        chatId: selectedChat.id,
                        senderId: Math.random() > 0.5 ? 'user1' : 'user2', // Random sender for demo
                        content: `Simulated new message from ${Math.random() > 0.5 ? 'Alice' : 'Bob'} at ${new Date().toLocaleTimeString()}`,
                        timestamp: Date.now(),
                        isEdited: false,
                    };
                    if (newMockMessage.senderId !== currentUser.id) {
                        mockMessages.push(newMockMessage); // Update mock data globally
                        messages = [...mockMessages]; // Refresh local messages state
                        renderChatWindow(); // Re-render chat window to show new message
                        scrollToBottom();
                    }
                }, 15000); // New message every 15 seconds
            }
        }

        // --- Mock Backend Simulation Functions ---

        async function apiRegister(username, password) {
            const existingUser = mockUsers.find(u => u.username === username);
            if (existingUser) {
                showMessageBox('Username already exists. Please choose another.');
                return false;
            }
            const newUser = { id: `user${mockUsers.length + 1}`, username, password };
            mockUsers.push(newUser);
            showMessageBox('Registration successful! Please log in.');
            return true;
        }

        async function apiLogin(username, password) {
            const user = mockUsers.find(u => u.username === username && u.password === password);
            if (user) {
                isLoggedIn = true;
                currentUser = { id: user.id, username: user.username };
                // Filter chats that include the current user
                chats = mockChats.filter(chat => chat.participants.includes(currentUser.id));
                messages = [...mockMessages]; // Load all mock messages initially
                renderApp(); // Re-render the entire app
                setupMessageSimulation();
                return true;
            } else {
                showMessageBox('Invalid username or password.');
                return false;
            }
        }

        function apiLogout() {
            isLoggedIn = false;
            currentUser = null;
            selectedChat = null;
            chats = [];
            messages = [];
            if (messageInterval) {
                clearInterval(messageInterval);
            }
            showMessageBox('You have been logged out.');
            renderApp();
        }

        async function apiCreateChat(chatName, participantIds, isGroup) {
            const newChat = {
                id: `chat${Date.now()}`,
                name: chatName,
                isGroup: isGroup,
                participants: [...participantIds, currentUser.id], // Add current user to participants
            };
            chats.push(newChat); // Update local state
            mockChats.push(newChat); // Update mock data globally
            selectedChat = newChat; // Automatically select the new chat
            renderChatList(); // Re-render chat list
            renderChatWindow(); // Re-render chat window for the new chat
            setupMessageSimulation();
            return newChat;
        }

        async function apiDeleteChat(chatId) {
            chats = chats.filter(chat => chat.id !== chatId);
            messages = messages.filter(msg => msg.chatId !== chatId);
            mockChats = mockChats.filter(chat => chat.id !== chatId);
            mockMessages = mockMessages.filter(msg => msg.chatId !== chatId);
            if (selectedChat && selectedChat.id === chatId) {
                selectedChat = null;
            }
            showMessageBox('Chat deleted successfully.');
            renderChatList();
            renderChatWindow();
            setupMessageSimulation();
        }

        async function apiExitChat(chatId) {
            chats = chats.filter(chat => chat.id !== chatId);
            if (selectedChat && selectedChat.id === chatId) {
                selectedChat = null;
            }
            const chatToExit = mockChats.find(chat => chat.id === chatId);
            if (chatToExit) {
                chatToExit.participants = chatToExit.participants.filter(id => id !== currentUser.id);
            }
            showMessageBox('You have exited the chat.');
            renderChatList();
            renderChatWindow();
            setupMessageSimulation();
        }

        async function apiSendMessage(chatId, content) {
            const newMessage = {
                id: `msg${Date.now()}`,
                chatId: chatId,
                senderId: currentUser.id,
                content: content,
                timestamp: Date.now(),
                isEdited: false,
            };
            messages.push(newMessage); // Update local state
            mockMessages.push(newMessage); // Update mock data globally
            renderChatWindow(); // Re-render chat window
            scrollToBottom();
            return newMessage;
        }

        async function apiEditMessage(messageId, newContent) {
            messages = messages.map(msg =>
                msg.id === messageId ? { ...msg, content: newContent, isEdited: true, timestamp: Date.now() } : msg
            );
            const msgToEdit = mockMessages.find(msg => msg.id === messageId);
            if (msgToEdit) {
                msgToEdit.content = newContent;
                msgToEdit.isEdited = true;
                msgToEdit.timestamp = Date.now();
            }
            editingMessageId = null; // Clear editing state
            editingMessageContent = '';
            renderChatWindow(); // Re-render chat window
        }

        async function apiDeleteMessage(messageId) {
            messages = messages.filter(msg => msg.id !== messageId);
            mockMessages = mockMessages.filter(msg => msg.id !== messageId);
            renderChatWindow(); // Re-render chat window
        }

        // --- Components as Functions (return DOM elements) ---

        function AuthComponent() {
            const authDiv = document.createElement('div');
            authDiv.className = "min-h-screen flex items-center justify-center bg-gray-100 p-4";
            authDiv.innerHTML = `
                <div class="bg-white p-8 rounded-xl shadow-2xl w-full max-w-md">
                    <h2 id="auth-title" class="text-3xl font-bold mb-6 text-center text-gray-800">
                        ${showRegister ? 'Register' : 'Login'}
                    </h2>
                    <form id="auth-form">
                        <div class="mb-4">
                            <label class="block text-gray-700 text-sm font-medium mb-2" htmlFor="username">
                                Username
                            </label>
                            <input
                                type="text"
                                id="auth-username"
                                class="shadow-sm appearance-none border rounded-lg w-full py-3 px-4 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200"
                                required
                            />
                        </div>
                        <div class="mb-6">
                            <label class="block text-gray-700 text-sm font-medium mb-2" htmlFor="password">
                                Password
                            </label>
                            <input
                                type="password"
                                id="auth-password"
                                class="shadow-sm appearance-none border rounded-lg w-full py-3 px-4 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200"
                                required
                            />
                        </div>
                        <button
                            type="submit"
                            class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-lg focus:outline-none focus:shadow-outline transition duration-300 transform hover:scale-105"
                        >
                            ${showRegister ? 'Register' : 'Login'}
                        </button>
                    </form>
                    <div class="mt-6 text-center">
                        <button
                            id="toggle-auth-mode"
                            class="text-blue-600 hover:text-blue-800 font-medium transition duration-200"
                        >
                            ${showRegister ? 'Already have an account? Login' : 'Need an account? Register'}
                        </button>
                    </div>
                </div>
            `;

            const authForm = authDiv.querySelector('#auth-form');
            authForm.addEventListener('submit', async (e) => {
                e.preventDefault();
                const username = authDiv.querySelector('#auth-username').value;
                const password = authDiv.querySelector('#auth-password').value;
                if (showRegister) {
                    await apiRegister(username, password);
                } else {
                    await apiLogin(username, password);
                }
            });

            authDiv.querySelector('#toggle-auth-mode').addEventListener('click', () => {
                showRegister = !showRegister;
                renderApp(); // Re-render to update UI
            });

            return authDiv;
        }

        function MainLayout() {
            const mainDiv = document.createElement('div');
            mainDiv.className = "flex flex-1 h-screen";
            mainDiv.innerHTML = `
                <div id="chat-list-container" class="w-1/4 max-w-xs flex flex-col border-r border-gray-200 bg-gray-50 shadow-lg">
                    <!-- ChatList content will be rendered here -->
                </div>
                <div id="chat-window-container" class="flex-1 flex flex-col">
                    <!-- ChatWindow content will be rendered here -->
                </div>
            `;
            // Render sub-components
            renderChatList();
            renderChatWindow();
            return mainDiv;
        }

        function renderChatList() {
            const container = document.getElementById('chat-list-container');
            if (!container) return; // Exit if container doesn't exist (e.g., not logged in)

            container.innerHTML = `
                <div class="p-4 border-b border-gray-200 bg-white flex justify-between items-center shadow-sm">
                    <h2 class="text-xl font-bold text-gray-800">
                        Welcome, ${currentUser?.username}!
                    </h2>
                    <button id="logout-btn"
                        class="p-2 bg-red-500 text-white rounded-full shadow-md hover:bg-red-600 transition duration-200"
                        title="Logout"
                    >
                        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-log-out"><path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"/><polyline points="16 17 21 12 16 7"/><line x1="21" x2="9" y1="12" y2="12"/></svg>
                    </button>
                </div>
                <div id="chats-list" class="flex-1 overflow-y-auto">
                    ${chats.length === 0 ? `
                        <p class="p-4 text-gray-500 text-center">No chats yet. Create one!</p>
                    ` : ''}
                </div>
                <div id="create-chat-modal-container"></div>
            `;

            container.querySelector('#logout-btn').addEventListener('click', apiLogout);

            const chatsListDiv = container.querySelector('#chats-list');
            chats.forEach(chat => {
                const chatDiv = document.createElement('div');
                chatDiv.className = "flex items-center justify-between p-4 border-b border-gray-100 hover:bg-blue-50 cursor-pointer transition duration-150";
                chatDiv.dataset.chatId = chat.id;

                const getChatDisplayName = (c) => {
                    if (c.isGroup) {
                        return c.name;
                    } else {
                        const otherParticipant = c.participants.find(id => id !== currentUser.id);
                        const otherUser = mockUsers.find(u => u.id === otherParticipant);
                        return otherUser ? otherUser.username : 'Unknown User';
                    }
                };

                chatDiv.innerHTML = `
                    <div class="flex items-center">
                        ${chat.isGroup ?
                            `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-users text-gray-600 mr-3"><path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M22 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>` :
                            `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-user-plus text-gray-600 mr-3"><path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><line x1="19" x2="19" y1="8" y2="14"/><line x1="22" x2="16" y1="11" y2="11"/></svg>`
                        }
                        <span class="font-medium text-gray-800 truncate">${getChatDisplayName(chat)}</span>
                    </div>
                    <div class="flex space-x-2">
                        <button data-action="delete"
                            class="p-1 text-red-500 hover:bg-red-100 rounded-full transition duration-150"
                            title="Delete Chat"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-trash-2"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
                        </button>
                        <button data-action="exit"
                            class="p-1 text-yellow-600 hover:bg-yellow-100 rounded-full transition duration-150"
                            title="Exit Chat"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-log-out"><path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"/><polyline points="16 17 21 12 16 7"/><line x1="21" x2="9" y1="12" y2="12"/></svg>
                        </button>
                    </div>
                `;
                chatDiv.addEventListener('click', () => {
                    selectedChat = chat;
                    renderChatWindow();
                    setupMessageSimulation();
                });
                chatDiv.querySelector('[data-action="delete"]').addEventListener('click', (e) => {
                    e.stopPropagation();
                    apiDeleteChat(chat.id);
                });
                chatDiv.querySelector('[data-action="exit"]').addEventListener('click', (e) => {
                    e.stopPropagation();
                    apiExitChat(chat.id);
                });
                chatsListDiv.appendChild(chatDiv);
            });

            // Add Create Chat button
            const createChatBtn = document.createElement('button');
            createChatBtn.className = "p-2 bg-blue-500 text-white rounded-full shadow-md hover:bg-blue-600 transition duration-200 absolute bottom-4 right-4";
            createChatBtn.title = "Create New Chat";
            createChatBtn.innerHTML = `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-user-plus"><path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><line x1="19" x2="19" y1="8" y2="14"/><line x1="22" x2="16" y1="11" y2="11"/></svg>`;
            createChatBtn.addEventListener('click', () => {
                showCreateChatModal = true;
                renderCreateChatModal();
            });
            container.appendChild(createChatBtn);

            if (showCreateChatModal) {
                renderCreateChatModal();
            } else {
                 const modalContainer = document.getElementById('create-chat-modal-container');
                 if (modalContainer) modalContainer.innerHTML = '';
            }
        }

        function renderCreateChatModal() {
            const modalContainer = document.getElementById('create-chat-modal-container');
            if (!modalContainer) return;

            modalContainer.innerHTML = `
                <div class="fixed inset-0 bg-gray-800 bg-opacity-75 flex items-center justify-center z-50 p-4">
                    <div class="bg-white rounded-xl shadow-2xl p-6 w-full max-w-md relative">
                        <button id="close-create-chat-modal"
                            class="absolute top-3 right-3 text-gray-500 hover:text-gray-800"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
                        </button>
                        <h3 class="text-2xl font-bold mb-5 text-gray-800 text-center">Create New Chat</h3>

                        <div class="flex justify-center mb-5 border-b border-gray-200">
                            <button id="one-on-one-tab"
                                class="py-2 px-4 rounded-t-lg font-semibold ${createChatType === 'one-on-one' ? 'bg-blue-600 text-white' : 'text-blue-600 bg-gray-100 hover:bg-gray-200'}"
                            >
                                One-on-One
                            </button>
                            <button id="group-tab"
                                class="py-2 px-4 rounded-t-lg font-semibold ${createChatType === 'group' ? 'bg-blue-600 text-white' : 'text-blue-600 bg-gray-100 hover:bg-gray-200'}"
                            >
                                Group Chat
                            </button>
                        </div>

                        <div id="chat-creation-content"></div>
                    </div>
                </div>
            `;

            modalContainer.querySelector('#close-create-chat-modal').addEventListener('click', () => {
                showCreateChatModal = false;
                renderChatList(); // Re-render chat list to close modal
            });
            modalContainer.querySelector('#one-on-one-tab').addEventListener('click', () => {
                createChatType = 'one-on-one';
                renderCreateChatModal();
            });
            modalContainer.querySelector('#group-tab').addEventListener('click', () => {
                createChatType = 'group';
                renderCreateChatModal();
            });

            const chatCreationContent = modalContainer.querySelector('#chat-creation-content');
            if (createChatType === 'one-on-one') {
                const otherUsers = mockUsers.filter(u => u.id !== currentUser.id && !chats.some(chat => !chat.isGroup && chat.participants.includes(u.id) && chat.participants.includes(currentUser.id)));
                chatCreationContent.innerHTML = `
                    <div>
                        <h4 class="text-lg font-semibold mb-3 text-gray-700">Select User:</h4>
                        <div class="max-h-60 overflow-y-auto border border-gray-300 rounded-lg p-2">
                            ${otherUsers.length === 0 ? `
                                <p class="text-gray-500">No other users available to start a new chat with.</p>
                            ` : otherUsers.map(user => `
                                <button
                                    data-user-id="${user.id}"
                                    class="w-full text-left py-2 px-3 rounded-lg hover:bg-blue-100 transition duration-150 mb-1 last:mb-0"
                                >
                                    ${user.username}
                                </button>
                            `).join('')}
                        </div>
                    </div>
                `;
                otherUsers.forEach(user => {
                    chatCreationContent.querySelector(`[data-user-id="${user.id}"]`).addEventListener('click', async () => {
                         // Check if chat already exists
                        const existingChat = chats.find(
                            chat => !chat.isGroup &&
                            chat.participants.includes(currentUser.id) &&
                            chat.participants.includes(user.id)
                        );

                        if (existingChat) {
                            showMessageBox(`Chat with ${user.username} already exists!`);
                            selectedChat = existingChat;
                            showCreateChatModal = false;
                            renderChatList();
                            renderChatWindow();
                            return;
                        }

                        await apiCreateChat(`${currentUser.username} & ${user.username}`, [user.id], false);
                        showCreateChatModal = false;
                        renderChatList(); // Re-render to close modal
                    });
                });
            } else { // Group Chat
                const allOtherUsers = mockUsers.filter(u => u.id !== currentUser.id);
                chatCreationContent.innerHTML = `
                    <div>
                        <div class="mb-4">
                            <label class="block text-gray-700 text-sm font-medium mb-2" htmlFor="groupName">
                                Group Name
                            </label>
                            <input
                                type="text"
                                id="groupName"
                                class="shadow-sm appearance-none border rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200"
                                placeholder="Enter group name"
                                required
                            />
                        </div>
                        <h4 class="text-lg font-semibold mb-3 text-gray-700">Select Participants:</h4>
                        <div id="participant-list" class="max-h-60 overflow-y-auto border border-gray-300 rounded-lg p-2">
                            ${allOtherUsers.map(user => `
                                <div class="flex items-center mb-2">
                                    <input
                                        type="checkbox"
                                        id="user-checkbox-${user.id}"
                                        value="${user.id}"
                                        class="form-checkbox h-5 w-5 text-blue-600 rounded focus:ring-blue-500"
                                    />
                                    <label htmlFor="user-checkbox-${user.id}" class="ml-2 text-gray-700">
                                        ${user.username}
                                    </label>
                                </div>
                            `).join('')}
                        </div>
                        <button id="create-group-btn"
                            class="mt-5 w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg focus:outline-none focus:shadow-outline transition duration-300 transform hover:scale-105"
                        >
                            Create Group
                        </button>
                    </div>
                `;

                const selectedParticipants = [];
                chatCreationContent.querySelectorAll('#participant-list input[type="checkbox"]').forEach(checkbox => {
                    checkbox.addEventListener('change', (e) => {
                        const userId = e.target.value;
                        if (e.target.checked) {
                            selectedParticipants.push(userId);
                        } else {
                            const index = selectedParticipants.indexOf(userId);
                            if (index > -1) {
                                selectedParticipants.splice(index, 1);
                            }
                        }
                    });
                });

                chatCreationContent.querySelector('#create-group-btn').addEventListener('click', async () => {
                    const groupNameInput = chatCreationContent.querySelector('#groupName');
                    const groupName = groupNameInput.value.trim();

                    if (!groupName) {
                        showMessageBox('Group name cannot be empty.');
                        return;
                    }
                    if (selectedParticipants.length === 0) {
                        showMessageBox('Please select at least one participant for the group.');
                        return;
                    }
                    await apiCreateChat(groupName, selectedParticipants, true);
                    showCreateChatModal = false;
                    renderChatList(); // Re-render to close modal
                });
            }
        }

        function renderChatWindow() {
            const container = document.getElementById('chat-window-container');
            if (!container) return; // Exit if container doesn't exist (e.g., not logged in)

            container.innerHTML = ''; // Clear existing content

            if (!selectedChat) {
                container.innerHTML = `
                    <div class="flex-1 flex items-center justify-center bg-gray-100 text-gray-500 text-lg">
                        Select a chat to start messaging!
                    </div>
                `;
                return;
            }

            const getSenderUsername = (senderId) => {
                const user = mockUsers.find(u => u.id === senderId);
                return user ? user.username : 'Unknown';
            };

            const formatTimestamp = (timestamp) => {
                const date = new Date(timestamp);
                return date.toLocaleString('en-US', { hour: 'numeric', minute: 'numeric', hour12: true });
            };

            const filteredMessages = messages.filter(msg => msg.chatId === selectedChat.id);

            container.innerHTML = `
                <div class="flex flex-col h-full bg-gray-100">
                    <div class="p-4 border-b border-gray-200 bg-white shadow-sm">
                        <h3 class="text-xl font-bold text-gray-800">
                            ${selectedChat.isGroup ? selectedChat.name : getSenderUsername(selectedChat.participants.find(id => id !== currentUser.id))}
                        </h3>
                        <p class="text-sm text-gray-500">
                            ${selectedChat.isGroup ? `Participants: ${selectedChat.participants.map(id => getSenderUsername(id)).join(', ')}` : ''}
                        </p>
                    </div>

                    <div id="messages-display" class="flex-1 overflow-y-auto p-4 space-y-4">
                        ${filteredMessages.length === 0 ? `
                            <p class="text-gray-500 text-center py-4">No messages in this chat yet. Say hello!</p>
                        ` : ''}
                    </div>

                    <form id="message-form" class="p-4 border-t border-gray-200 bg-white shadow-lg flex items-center">
                        <input
                            type="text"
                            id="new-message-content"
                            class="flex-1 p-3 border border-gray-300 rounded-full focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent mr-3"
                            placeholder="Type a message..."
                            ${!currentUser ? 'disabled' : ''}
                        />
                        <button
                            type="submit"
                            id="send-message-btn"
                            class="p-3 bg-blue-600 text-white rounded-full shadow-md hover:bg-blue-700 transition duration-300 transform hover:scale-105 disabled:opacity-50 disabled:cursor-not-allowed"
                            ${!currentUser ? 'disabled' : ''}
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-send"><path d="m22 2-7 7m0 0-7 7m7-7L2 22"/><path d="M11 2H9a2 2 0 0 0-2 2v2"/><path d="M13 22h2a2 2 0 0 0 2-2v-2"/></svg>
                        </button>
                    </form>
                </div>
            `;

            const messagesDisplay = container.querySelector('#messages-display');
            filteredMessages.forEach(message => {
                const messageDiv = document.createElement('div');
                messageDiv.className = `flex ${message.senderId === currentUser.id ? 'justify-end' : 'justify-start'}`;
                messageDiv.classList.add('group'); // Add group class for hover effects

                const messageBubble = document.createElement('div');
                messageBubble.className = `relative p-3 rounded-lg shadow-md max-w-[75%] ${
                    message.senderId === currentUser.id
                        ? 'bg-blue-600 text-white rounded-br-none'
                        : 'bg-white text-gray-800 rounded-bl-none border border-gray-200'
                }`;

                if (message.senderId !== currentUser.id) {
                    messageBubble.innerHTML += `
                        <p class="font-semibold text-sm mb-1 ${message.senderId === currentUser.id ? 'text-blue-200' : 'text-gray-600'}">
                            ${getSenderUsername(message.senderId)}
                        </p>
                    `;
                }

                if (editingMessageId === message.id) {
                    messageBubble.innerHTML += `
                        <div class="flex flex-col">
                            <textarea
                                id="editing-message-textarea-${message.id}"
                                class="w-full p-2 rounded-md bg-white text-gray-800 border border-gray-300 resize-none"
                                rows="3"
                            >${editingMessageContent}</textarea>
                            <div class="flex justify-end mt-2 space-x-2">
                                <button data-action="save-edit" data-message-id="${message.id}"
                                    class="p-2 bg-green-500 text-white rounded-full hover:bg-green-600 transition duration-200"
                                    title="Save Edit"
                                >
                                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-save"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/><polyline points="17 21 17 13 7 13 7 21"/><polyline points="7 3 7 8 15 8"/></svg>
                                </button>
                                <button data-action="cancel-edit"
                                    class="p-2 bg-gray-500 text-white rounded-full hover:bg-gray-600 transition duration-200"
                                    title="Cancel Edit"
                                >
                                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
                                </button>
                            </div>
                        </div>
                    `;
                     // Add event listeners for edit actions within the messageBubble context
                    messageBubble.querySelector('[data-action="save-edit"]').addEventListener('click', async () => {
                        const newContent = messageBubble.querySelector(`#editing-message-textarea-${message.id}`).value;
                        if (newContent.trim()) {
                            await apiEditMessage(message.id, newContent);
                        } else {
                            showMessageBox('Message cannot be empty.');
                        }
                    });
                    messageBubble.querySelector('[data-action="cancel-edit"]').addEventListener('click', () => {
                        editingMessageId = null;
                        editingMessageContent = '';
                        renderChatWindow();
                    });

                } else {
                    messageBubble.innerHTML += `<p class="text-base break-words">${message.content}</p>`;
                }

                messageBubble.innerHTML += `
                    <div class="text-xs mt-1 flex items-center ${message.senderId === currentUser.id ? 'text-blue-200 justify-end' : 'text-gray-400 justify-start'}">
                        ${message.isEdited ? `<span class="mr-1">(edited)</span>` : ''}
                        <span>${formatTimestamp(message.timestamp)}</span>
                    </div>
                `;

                if (message.senderId === currentUser.id && editingMessageId !== message.id) {
                    messageBubble.innerHTML += `
                        <div class="absolute top-1 right-1 flex space-x-1 opacity-0 group-hover:opacity-100 transition-opacity duration-200">
                            <button data-action="edit" data-message-id="${message.id}"
                                class="p-1 text-white bg-blue-500 rounded-full hover:bg-blue-600 transition duration-150"
                                title="Edit Message"
                            >
                                <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-edit"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/></svg>
                            </button>
                            <button data-action="delete" data-message-id="${message.id}"
                                class="p-1 text-white bg-red-500 rounded-full hover:bg-red-600 transition duration-150"
                                title="Delete Message"
                            >
                                <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-trash-2"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
                            </button>
                        </div>
                    `;
                    messageBubble.querySelector('[data-action="edit"]').addEventListener('click', (e) => {
                        editingMessageId = message.id;
                        editingMessageContent = message.content;
                        renderChatWindow();
                    });
                    messageBubble.querySelector('[data-action="delete"]').addEventListener('click', (e) => {
                        apiDeleteMessage(message.id);
                    });
                }
                messageDiv.appendChild(messageBubble);
                messagesDisplay.appendChild(messageDiv);
            });

            // Auto-scrolling element
            messagesEndRef = document.createElement('div');
            messagesDisplay.appendChild(messagesEndRef);
            scrollToBottom();

            const messageForm = container.querySelector('#message-form');
            messageForm.addEventListener('submit', async (e) => {
                e.preventDefault();
                const newMessageInput = container.querySelector('#new-message-content');
                const content = newMessageInput.value;
                if (content.trim()) {
                    await apiSendMessage(selectedChat.id, content);
                    newMessageInput.value = '';
                }
            });
        }

        // Initialize the app on window load
        window.onload = function() {
            renderApp();
        };
    </script>
</body>
</html>
