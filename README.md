<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Log Viewer</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 20px;
            background-color: #f4f4f4;
        }
        h1 {
            text-align: center;
            color: #333;
        }
        .search-container {
            margin-bottom: 20px;
            display: flex;
            justify-content: center;
            gap: 20px;
        }
        .search-container input {
            padding: 8px;
            width: 200px;
            font-size: 16px;
            border-radius: 4px;
            border: 1px solid #ddd;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 8px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .timestamp {
            width: 150px;
        }
        .classname {
            width: 200px;
        }
        .message {
            width: 100%;
        }
        .highlight {
            background-color: yellow;
            font-weight: bold;
            cursor: pointer;
        }
    </style>
</head>
<body>

<h1>Log Viewer</h1>

<div class="search-container">
    <input type="text" id="search-timestamp" placeholder="Search by Timestamp" onkeyup="highlightAndScroll()">
    <input type="text" id="search-classname" placeholder="Search by Class Name" onkeyup="highlightAndScroll()">
    <input type="text" id="search-message" placeholder="Search by Message" onkeyup="highlightAndScroll()">
</div>

<table>
    <thead>
        <tr>
            <th class="timestamp">Timestamp</th>
            <th class="classname">Class Name</th>
            <th class="message">Message</th>
        </tr>
    </thead>
    <tbody id="log-table-body">
        <!-- Log entries will be inserted here dynamically -->
    </tbody>
</table>

<script>
// Function to highlight matching text in a string
function highlightText(text, searchQuery) {
    if (!searchQuery) return text; // Return the original text if there's no search query

    const regex = new RegExp(`(${searchQuery})`, 'gi');
    return text.replace(regex, '<span class="highlight">$1</span>');
}

// Function to populate the table with logs
function populateLogTable(logs) {
    const tableBody = document.getElementById('log-table-body');
    tableBody.innerHTML = ''; // Clear any existing rows

    logs.forEach(log => {
        const row = document.createElement('tr');
        
        const timestampCell = document.createElement('td');
        timestampCell.classList.add('timestamp');
        timestampCell.innerHTML = highlightText(log.timestamp, document.getElementById('search-timestamp').value);
        
        const classNameCell = document.createElement('td');
        classNameCell.classList.add('classname');
        classNameCell.innerHTML = highlightText(log.className, document.getElementById('search-classname').value);
        
        const messageCell = document.createElement('td');
        messageCell.classList.add('message');
        messageCell.innerHTML = highlightText(log.message, document.getElementById('search-message').value);
        
        row.appendChild(timestampCell);
        row.appendChild(classNameCell);
        row.appendChild(messageCell);
        
        tableBody.appendChild(row);
    });
}

// Function to highlight text and scroll to the highlighted word
function highlightAndScroll() {
    populateLogTable(logs); // Re-populate the table with highlighted text

    // Scroll to the first highlighted element if it exists
    const highlightedElements = document.querySelectorAll('.highlight');
    if (highlightedElements.length > 0) {
        highlightedElements[0].scrollIntoView({
            behavior: 'smooth',
            block: 'center'
        });
    }
}

// Fetch logs from an API
async function fetchLogs() {
    try {
        const response = await fetch('https://your-api-endpoint.com/logs'); // Replace with your API URL
        const data = await response.json();
        logs = data; // Store the fetched data globally
        populateLogTable(logs); // Populate the table with the fetched data
    } catch (error) {
        console.error('Error fetching log data:', error);
    }
}

// Initializing logs variable
let logs = [];

// Fetch logs on page load
fetchLogs();
</script>

</body>
</html>
