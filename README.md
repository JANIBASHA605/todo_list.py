# todo_list.py
from flask import Flask, request, jsonify, render_template

app = Flask(__name__)

# In-memory task list
tasks = []

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/tasks', methods=['GET', 'POST'])
def manage_tasks():
    if request.method == 'POST':
        data = request.json
        task = {
            'id': len(tasks) + 1,
            'title': data['title'],
            'completed': False
        }
        tasks.append(task)
        return jsonify(task), 201
    
    # Filtering and searching
    search_query = request.args.get('search', '').lower()
    filtered_tasks = [task for task in tasks if search_query in task['title'].lower()]
    return jsonify(filtered_tasks)

@app.route('/tasks/<int:task_id>', methods=['PUT', 'DELETE'])
def update_task(task_id):
    global tasks
    task = next((task for task in tasks if task['id'] == task_id), None)
    if not task:
        return jsonify({'error': 'Task not found'}), 404
    
    if request.method == 'PUT':
        task['completed'] = not task['completed']
        return jsonify(task)
    
    if request.method == 'DELETE':
        tasks = [task for task in tasks if task['id'] != task_id]
        return '', 204

if __name__ == '__main__':
    app.run(debug=True)

# index.html (Frontend)
with open('templates/index.html', 'w') as f:
    f.write('''
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>To-Do List</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; }
            input, button { margin: 5px; padding: 10px; }
            .completed { text-decoration: line-through; color: gray; }
        </style>
    </head>
    <body>
        <h1>To-Do List</h1>
        <input type="text" id="taskInput" placeholder="New task...">
        <button onclick="addTask()">Add Task</button>
        <input type="text" id="searchInput" placeholder="Search tasks..." oninput="fetchTasks()">
        <ul id="taskList"></ul>

        <script>
            async function fetchTasks() {
                const searchQuery = document.getElementById('searchInput').value;
                const response = await fetch(`/tasks?search=${encodeURIComponent(searchQuery)}`);
                const tasks = await response.json();
                const taskList = document.getElementById('taskList');
                taskList.innerHTML = '';
                tasks.forEach(task => {
                    const li = document.createElement('li');
                    li.textContent = task.title;
                    if (task.completed) li.classList.add('completed');
                    li.onclick = () => toggleTask(task.id);
                    const deleteBtn = document.createElement('button');
                    deleteBtn.textContent = 'Delete';
                    deleteBtn.onclick = (e) => { e.stopPropagation(); deleteTask(task.id); };
                    li.appendChild(deleteBtn);
                    taskList.appendChild(li);
                });
            }

            async function addTask() {
                const taskInput = document.getElementById('taskInput');
                const title = taskInput.value.trim();
                if (!title) return;
                await fetch('/tasks', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ title })
                });
                taskInput.value = '';
                fetchTasks();
            }

            async function toggleTask(id) {
                await fetch(`/tasks/${id}`, { method: 'PUT' });
                fetchTasks();
            }

            async function deleteTask(id) {
                await fetch(`/tasks/${id}`, { method: 'DELETE' });
                fetchTasks();
            }

            window.onload = fetchTasks;
        </script>
    </body>
    </html>
    ''')
