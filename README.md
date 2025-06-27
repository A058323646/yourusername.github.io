<!DOCTYPE html>
<html lang="he" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>מערכת וואטסאפ לרכב</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #222;
            color: #eee;
            text-align: center;
            padding: 20px;
            margin: 0;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
        }
        #message-display {
            font-size: 2em;
            margin-bottom: 20px;
            min-height: 100px;
            display: flex;
            align-items: center;
            justify-content: center;
            background-color: #333;
            border-radius: 10px;
            padding: 15px;
            width: 90%;
            max-width: 600px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
            text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
        }
        #status-display {
            font-size: 1.2em;
            color: #bbb;
            margin-bottom: 30px;
        }
        button {
            background-color: #007bff;
            color: white;
            border: none;
            padding: 15px 30px;
            font-size: 1.5em;
            border-radius: 8px;
            cursor: pointer;
            margin: 10px;
            transition: background-color 0.3s ease;
            width: 80%;
            max-width: 400px;
        }
        button:hover {
            background-color: #0056b3;
        }
        button:active {
            background-color: #004080;
        }
        #reply-input {
            width: 90%;
            max-width: 600px;
            padding: 10px;
            font-size: 1.2em;
            border: 1px solid #555;
            border-radius: 5px;
            background-color: #444;
            color: #eee;
            margin-top: 20px;
        }
        #speak-button, #send-reply-button, #answer-button {
             display: none; /* ננהל את התצוגה שלהם דינמית */
        }
        #main-content {
            display: none;
            flex-direction: column;
            align-items: center;
            width: 100%;
        }
        #start-button {
            background-color: #28a745;
            margin-top: 50px;
        }
        #start-button:hover {
            background-color: #218838;
        }
    </style>
</head>
<body>
    <h1>מערכת וואטסאפ לרכב</h1>

    <button id="start-button">התחל מערכת</button>

    <div id="main-content">
        <div id="status-display">ממתין להודעות...</div>
        <div id="message-display"></div>

        <button id="answer-button">האם לשלוח תשובה?</button>
        <button id="speak-button">התחל הקלדה קולית</button>
        <button id="send-reply-button">שלח תשובה</button>
        <input type="text" id="reply-input" placeholder="הקלד או אמור את התשובה...">
    </div>

    <script>
        const GOOGLE_APP_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbxFZHY9VzIZRRoudK41hP4QzAlSAKV72AMnAC_1Qj_ur6lcbTtj9-B6aOd9rtRjFKQ5hA/exec';

        let lastReadMessage = "";
        let isSpeaking = false;
        let expectingYesNo = false; // NEW: Flag to indicate we are expecting a "yes" or "no"
        
        const messageDisplay = document.getElementById('message-display');
        const statusDisplay = document.getElementById('status-display');
        const answerButton = document.getElementById('answer-button'); // This button is now largely redundant for voice input
        const speakButton = document.getElementById('speak-button');
        const sendReplyButton = document.getElementById('send-reply-button');
        const replyInput = document.getElementById('reply-input');

        const startButton = document.getElementById('start-button');
        const mainContent = document.getElementById('main-content');

        // Speech Synthesis (Text-to-Speech)
        const synth = window.speechSynthesis;
        const utterance = new SpeechSynthesisUtterance();
        utterance.lang = 'he-IL';

        utterance.onend = () => {
            isSpeaking = false;
            statusDisplay.textContent = "הקראה הסתיימה.";
            // NEW: After reading the main message, ask if user wants to reply
            if (!expectingYesNo && messageDisplay.textContent && 
                messageDisplay.textContent !== "אין הודעות חדשות" && 
                messageDisplay.textContent !== "אין הודעות חדשות לקריאה.") {
                
                // Prompt the user for a reply decision
                readText("האם תרצה להחזיר תשובה?", true, true); // The last 'true' flag means this is a "yes/no" question
            } else if (expectingYesNo) { // NEW: If we just finished asking "yes/no", start listening
                expectingYesNo = false; // Reset flag
                if (recognition) {
                    statusDisplay.textContent = "ממתין לתשובת כן/לא...";
                    recognition.start(); // Start listening for "yes" or "no"
                }
            }
        };
        utterance.onerror = (event) => {
            console.error("Speech synthesis error:", event.error);
            statusDisplay.textContent = "שגיאה בהקראה קולית. ייתכן שנדרשת אינטראקציה עם הדף.";
            isSpeaking = false;
        };

        // Speech Recognition (Voice-to-Text)
        const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
        let recognition = null;
        if (SpeechRecognition) {
            recognition = new SpeechRecognition();
            recognition.continuous = false;
            recognition.lang = 'he-IL';
            recognition.interimResults = false;

            recognition.onresult = (event) => {
                const transcript = event.results[0][0].transcript.trim(); // Trim whitespace
                statusDisplay.textContent = `זיהוי קולי: ${transcript}`;

                if (transcript.includes("כן") || transcript.includes("בטח") || transcript.includes("כן בטח")) { // NEW: Check for "yes" responses
                    statusDisplay.textContent = "ממתין לתשובה שלך...";
                    readText("אנא אמור את תשובתך.", true, false); // Ask for the actual reply
                    // Prepare for reply input
                    speakButton.style.display = 'block'; // Show "Start Voice Input" button
                    replyInput.style.display = 'block'; // Show text input field
                    sendReplyButton.style.display = 'block'; // Show "Send Reply" button
                    // Note: We don't start recognition immediately here, the user needs to press 'speakButton'
                    // for the actual reply input. Or you can add logic to start continuous listening.
                    // For now, let's keep it manual as originally designed for the actual reply.
                } else if (transcript.includes("לא") || transcript.includes("שלילי") || transcript.includes("לא תודה")) { // NEW: Check for "no" responses
                    statusDisplay.textContent = "לא נשלחת תשובה.";
                    readText("לא נשלחת תשובה.", true, false);
                    // Hide reply interface
                    answerButton.style.display = 'none';
                    speakButton.style.display = 'none';
                    sendReplyButton.style.display = 'none';
                    replyInput.style.display = 'none';
                    replyInput.value = '';
                    lastReadMessage = ""; // Reset for next message
                } else if (replyInput.style.display === 'block') { // NEW: If we're already expecting a full reply
                    replyInput.value = transcript;
                    readText(`התשובה שרשמת היא: ${transcript}. האם לשלוח?`, true, false); // Confirm the actual reply
                    sendReplyButton.style.display = 'block'; 
                } else { // NEW: If no clear "yes/no" or reply was given, retry
                    statusDisplay.textContent = "לא הבנתי. אנא אמור 'כן' או 'לא'.";
                    readText("לא הבנתי. אנא אמור 'כן' או 'לא'.", true, true); // Re-ask for yes/no
                }
            };

            recognition.onerror = (event) => {
                console.error("Speech recognition error:", event.error);
                statusDisplay.textContent = "שגיאה בזיהוי קולי. נסה שוב.";
                // Only show speak button if we're not in the yes/no phase
                if (!expectingYesNo) {
                    speakButton.style.display = 'block'; 
                }
            };

            recognition.onend = () => {
                // The recognition has ended. If we were expecting yes/no, and it wasn't captured, re-prompt or handle.
                // This 'onend' fires even if no speech was detected, so careful with auto-restarting.
                if (expectingYesNo) { // If it ended after a yes/no prompt and no valid input, re-prompt
                    // You might want a timeout here or a counter to prevent infinite loops
                    // For simplicity, we'll rely on the readText and onresult to guide flow
                }
            };
        } else {
            statusDisplay.textContent = "הדפדפן שלך אינו תומך בזיהוי קולי.";
            speakButton.style.display = 'none';
        }

        // NEW: added a 'isYesNoPrompt' parameter to differentiate between regular text and yes/no questions
        function readText(text, isReplyConfirmation = false, isYesNoPrompt = false) {
            if (synth.speaking) {
                synth.cancel(); 
            }
            if (text === "No new messages" || text === "No new unread messages" || text === "") {
                return;
            }

            utterance.text = text; // The text to speak

            if (isYesNoPrompt) {
                expectingYesNo = true; // Set flag when asking yes/no
            } else {
                expectingYesNo = false; // Clear flag for other types of speech
            }

            isSpeaking = true;
            synth.speak(utterance);
        }

        async function fetchMessages() {
            try {
                statusDisplay.textContent = "בודק הודעות חדשות...";
                const response = await fetch(`${GOOGLE_APP_SCRIPT_URL}?action=getLatestMessage`);
                const data = await response.json();
                
                if (data.status === "success" && data.message && data.message !== "No new messages" && data.message !== "No new unread messages") {
                    if (data.message !== lastReadMessage) {
                        messageDisplay.textContent = data.message;
                        lastReadMessage = data.message;
                        statusDisplay.textContent = "הודעה חדשה התקבלה.";
                        // The onend of this readText will trigger the "yes/no" question
                        readText(data.message); 
                    } else {
                        statusDisplay.textContent = "אין הודעות חדשות לקריאה.";
                    }
                } else {
                    messageDisplay.textContent = "אין הודעות חדשות"; 
                    statusDisplay.textContent = "אין הודעות חדשות.";
                }
            } catch (error) {
                console.error("Error fetching messages:", error);
                statusDisplay.textContent = "שגיאה בטעינת הודעות.";
            } finally {
                // Ensure all reply related UI is hidden initially
                answerButton.style.display = 'none';
                speakButton.style.display = 'none';
                sendReplyButton.style.display = 'none';
                replyInput.style.display = 'none';
                replyInput.value = ''; 
            }
        }

        async function sendReply() {
            const replyText = replyInput.value;
            if (!replyText) {
                statusDisplay.textContent = "הקלד או אמור תשובה לפני השליחה.";
                readText("אנא הקלד או אמור תשובה לפני השליחה.", true); // Voice prompt
                return;
            }

            try {
                statusDisplay.textContent = "שולח תשובה...";
                readText("שולח תשובה.", true); // Voice prompt
                const response = await fetch(GOOGLE_APP_SCRIPT_URL, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify({ type: "userReply", reply: replyText })
                });
                const result = await response.json();
                
                if (result.status === "success") {
                    statusDisplay.textContent = "התשובה נשלחה לגיליון בהצלחה! ממתין לשליחה בוואטסאפ.";
                    readText("התשובה נשלחה בהצלחה.", true); // Voice prompt
                } else {
                    statusDisplay.textContent = `שגיאה בשליחת התשובה: ${result.message}`;
                    readText(`שגיאה בשליחת התשובה: ${result.message}`, true); // Voice prompt
                }
                
                // Hide reply interface
                answerButton.style.display = 'none';
                speakButton.style.display = 'none';
                sendReplyButton.style.display = 'none';
                replyInput.style.display = 'none';
                replyInput.value = '';
                lastReadMessage = ""; 
            } catch (error) {
                console.error("Error sending reply:", error);
                statusDisplay.textContent = "שגיאה בשליחת התשובה.";
                readText("שגיאה בשליחת התשובה.", true); // Voice prompt
            }
        }

        let fetchIntervalId; 

        startButton.addEventListener('click', () => {
            startButton.style.display = 'none';
            mainContent.style.display = 'flex';
            
            fetchMessages();
            fetchIntervalId = setInterval(fetchMessages, 5000); 

            if (messageDisplay.textContent === "אין הודעות חדשות") {
                readText("המערכת הופעלה, ממתין להודעות.");
            }
        });

        document.addEventListener('DOMContentLoaded', () => {
            mainContent.style.display = 'none';
            startButton.style.display = 'block';
        });

        // The answerButton is now effectively only a visual prompt, the logic is in speech recognition
        answerButton.addEventListener('click', () => {
            // This button might become less relevant if all user interaction is voice based
            // For now, it could perhaps manually trigger the voice input for reply if yes/no failed
            statusDisplay.textContent = "אמור את תשובתך...";
            answerButton.style.display = 'none';
            speakButton.style.display = 'block'; // Still show manual speak button for actual reply
            if (recognition) {
                recognition.start(); // Start listening for the actual reply
            }
        });

        // The speakButton is for the actual reply
        speakButton.addEventListener('click', () => {
            statusDisplay.textContent = "אמור את תשובתך...";
            if (recognition) {
                recognition.start(); 
            }
        });

        sendReplyButton.addEventListener('click', sendReply);
    </script>
</body>
</html>
