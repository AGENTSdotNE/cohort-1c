# Adding a feature

This guide builds a small to-do list from start to finish: a Django model, an API, and a React page
that lists, creates, updates and deletes. Use the same shape for your own data, swapping `Task` for
whatever your feature manages.

Work through it in order. Each step builds on the one before, and you can check each one works before
moving on.

## The shape of a feature

```
backend/tasks/                  frontend/src/tasks/
  models.py       the table       api.ts            one function per endpoint
  serializers.py  JSON <-> model  use-tasks.ts      the query and a mutation helper
  views.py        the endpoints   tasks-page.tsx    the page
  urls.py         the URLs        add-task-form.tsx a form that creates
  admin.py        /admin/ screen  task-list.tsx     loading, error, empty and list states
  tests/          what must work  task-item.tsx     one row, which ticks off and deletes
```

Every feature gets its own Django app and its own frontend folder, named after the thing it manages.

## Backend

### 1. Start the app

```bash
cd backend
uv run python manage.py startapp tasks
rm tasks/tests.py
```

`startapp` makes a single `tests.py`, but this project keeps tests in a `tests/` folder, and Django
cannot have both.

Add `'tasks'` to `INSTALLED_APPS` in [config/settings.py](../backend/config/settings.py). In
[config/urls.py](../backend/config/urls.py), add its URLs to `urlpatterns`, next to the `api/auth/` line:

```python
path('api/', include('tasks.urls')),
```

### 2. The model

`tasks/models.py` is the table. The important line is `owner`: every row belongs to a user, and
`related_name='tasks'` means you can write `user.tasks.all()`.

```python
from django.conf import settings
from django.db import models


class Task(models.Model):
    owner = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='tasks')
    title = models.CharField(max_length=200)
    done = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['done', '-created_at']

    def __str__(self):
        return self.title
```

After changing a model, run both of these with `make dev` running, and commit the migration file it
creates. CI fails if a model change has no migration.

```bash
uv run python manage.py makemigrations
make migrate
```

### 3. The serializer

`tasks/serializers.py` turns a `Task` into JSON and back, and validates what the frontend sends. `owner`
is left out, so nobody can set it from the browser.

```python
from rest_framework import serializers

from .models import Task


class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ['id', 'title', 'done', 'created_at']
        read_only_fields = ['id', 'created_at']
```

### 4. The view

A `ModelViewSet` gives you list, create, update and delete in a few lines. Two methods do the security
work:

```python
from rest_framework import viewsets

from .serializers import TaskSerializer


class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    http_method_names = ['get', 'post', 'patch', 'delete']

    def get_queryset(self):
        return self.request.user.tasks.all()

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

- `get_queryset` returns only the signed-in user's tasks. Every action starts from it, so asking for
  someone else's task gets a 404, as if it did not exist.
- `perform_create` sets `owner` to the signed-in user.

**Forget `get_queryset` and every user can read and change everyone's data.** It is the one line to
check in every review.

You don't have to write a sign-in check. `IsAuthenticated` is the default for every view in
[settings.py](../backend/config/settings.py), so a signed-out request gets a 401 automatically.

### 5. The URLs

`tasks/urls.py` registers the viewset with a router:

```python
from rest_framework.routers import SimpleRouter

from .views import TaskViewSet

router = SimpleRouter()
router.register('tasks', TaskViewSet, basename='task')

urlpatterns = router.urls
```

That creates:

| Method | URL | Does |
| --- | --- | --- |
| GET | `/api/tasks/` | list your tasks |
| POST | `/api/tasks/` | create one |
| PATCH | `/api/tasks/<id>/` | change some fields |
| DELETE | `/api/tasks/<id>/` | delete one |

To see and edit tasks in Django admin, register the model in `tasks/admin.py` with
`admin.site.register(Task)`.

### 6. Tests

Test each endpoint, and above all test that you cannot touch someone else's data. Put them in
`tasks/tests/test_tasks.py` (with an empty `__init__.py` beside it) and run `make test`.

```python
from rest_framework import status
from rest_framework.test import APITestCase

from accounts.tests.factories import create_user

from ..models import Task


class TaskTests(APITestCase):
    def setUp(self):
        self.user = create_user()
        self.client.force_authenticate(self.user)

    def test_creating_a_task_makes_it_yours(self):
        response = self.client.post('/api/tasks/', {'title': 'Write the plan'}, format='json')

        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Task.objects.get().owner, self.user)

    def test_you_only_see_your_own_tasks(self):
        Task.objects.create(owner=self.user, title='Mine')
        Task.objects.create(owner=create_user(email='other@example.com'), title='Theirs')

        response = self.client.get('/api/tasks/')

        self.assertEqual([task['title'] for task in response.data], ['Mine'])

    def test_someone_elses_task_cannot_be_deleted(self):
        theirs = Task.objects.create(owner=create_user(email='other@example.com'), title='Theirs')

        response = self.client.delete(f'/api/tasks/{theirs.id}/')

        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)
```

## Frontend

### 7. The API functions

`src/tasks/api.ts` has a TypeScript type matching the serializer, and one small function per endpoint.
They all go through `apiRequest`, which adds the CSRF token and turns error responses into an
`ApiError`. Never call `fetch` directly.

```ts
import { apiRequest } from '@/lib/api-client'

export type Task = { id: number; title: string; done: boolean; created_at: string }

export const fetchTasks = () => apiRequest<Task[]>('/tasks/')

export const createTask = (title: string) => apiRequest<Task>('/tasks/', { body: { title } })

export const updateTask = (id: number, changes: Partial<Pick<Task, 'title' | 'done'>>) =>
  apiRequest<Task>(`/tasks/${id}/`, { method: 'PATCH', body: changes })

export const deleteTask = (id: number) => apiRequest<void>(`/tasks/${id}/`, { method: 'DELETE' })
```

### 8. Reading and changing data

`src/tasks/use-tasks.ts` wraps TanStack Query. `useTasks()` fetches the list and caches it under the key
`['tasks']`. Any component can call it, and they all share one request.

`useTaskMutation` wraps `useMutation`. When a change finishes, whether it worked or not, it invalidates
`['tasks']`, so the list refetches and the screen matches the server. You never update the list by hand.

```ts
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'

import { fetchTasks } from './api'

export const TASKS_KEY = ['tasks']

export const useTasks = () => useQuery({ queryKey: TASKS_KEY, queryFn: fetchTasks })

export const useTaskMutation = <TVariables>(mutationFn: (variables: TVariables) => Promise<unknown>) => {
  const queryClient = useQueryClient()

  return useMutation({ mutationFn, onSettled: () => queryClient.invalidateQueries({ queryKey: TASKS_KEY }) })
}
```

### 9. The list: handle every state

A query is always loading, failed, empty, or has data. Handle all four:

```tsx
import { Alert, AlertDescription } from '@/components/ui/alert'
import { Skeleton } from '@/components/ui/skeleton'

import { TaskItem } from './task-item'
import { useTasks } from './use-tasks'

export const TaskList = () => {
  const { data: tasks, isPending, isError, error } = useTasks()

  if (isPending) return <Skeleton className="h-24 w-full" />

  if (isError) {
    return (
      <Alert variant="destructive">
        <AlertDescription>{error.message}</AlertDescription>
      </Alert>
    )
  }

  if (tasks.length === 0) return <p className="text-muted-foreground text-sm">No tasks yet.</p>

  return (
    <ul className="divide-y rounded-md border">
      {tasks.map((task) => (
        <TaskItem key={task.id} task={task} />
      ))}
    </ul>
  )
}
```

### 10. A form that creates

Forms use react-hook-form with a zod schema, and `useApiForm`, which puts any error from Django back onto
the right field. The account settings forms in [src/account](../frontend/src/account) work the same way.

```tsx
import { zodResolver } from '@hookform/resolvers/zod'
import { useForm } from 'react-hook-form'
import { z } from 'zod'

import { FormField } from '@/components/form-field'
import { Alert, AlertDescription } from '@/components/ui/alert'
import { Button } from '@/components/ui/button'
import { useApiForm } from '@/lib/use-api-form'

import { createTask } from './api'
import { useTaskMutation } from './use-tasks'

const schema = z.object({ title: z.string().trim().min(1, 'Give the task a title') })

type Values = z.infer<typeof schema>

export const AddTaskForm = () => {
  const form = useForm<Values>({ resolver: zodResolver(schema), defaultValues: { title: '' } })
  const create = useTaskMutation(createTask)

  const { submit, error, isSubmitting } = useApiForm(form, async ({ title }) => {
    await create.mutateAsync(title)
    form.reset()
  })

  return (
    <form onSubmit={submit} noValidate className="grid gap-3">
      <FormField label="New task" error={form.formState.errors.title} registration={form.register('title')} />

      {error && (
        <Alert variant="destructive">
          <AlertDescription>{error}</AlertDescription>
        </Alert>
      )}

      <Button type="submit" disabled={isSubmitting} className="justify-self-start">
        Add task
      </Button>
    </form>
  )
}
```

### 11. Buttons that update and delete

Mutations live in the component that has the buttons. Show failures with a toast:

```tsx
import { Check, Trash2 } from 'lucide-react'
import { toast } from 'sonner'

import { Button } from '@/components/ui/button'
import { cn } from '@/lib/utils'

import { deleteTask, type Task, updateTask } from './api'
import { useTaskMutation } from './use-tasks'

const showError = (error: Error) => toast.error(error.message)

export const TaskItem = ({ task }: { task: Task }) => {
  const toggle = useTaskMutation(() => updateTask(task.id, { done: !task.done }))
  const remove = useTaskMutation(() => deleteTask(task.id))

  return (
    <li className="flex items-center gap-3 px-4 py-2">
      <Button
        variant="outline"
        size="icon"
        aria-label={task.done ? `Mark ${task.title} as not done` : `Mark ${task.title} as done`}
        disabled={toggle.isPending}
        onClick={() => toggle.mutate(undefined, { onError: showError })}
      >
        {task.done && <Check />}
      </Button>
      <span className={cn('flex-1 text-sm', task.done && 'text-muted-foreground line-through')}>{task.title}</span>
      <Button
        variant="ghost"
        size="icon"
        aria-label={`Delete ${task.title}`}
        disabled={remove.isPending}
        onClick={() => remove.mutate(undefined, { onError: showError })}
      >
        <Trash2 />
      </Button>
    </li>
  )
}
```

### 12. The page, route and link

The page just lays out the form and the list:

```tsx
import { AddTaskForm } from './add-task-form'
import { TaskList } from './task-list'

export const TasksPage = () => (
  <div className="mx-auto grid max-w-2xl gap-6">
    <h1 className="text-2xl font-semibold">Tasks</h1>
    <AddTaskForm />
    <TaskList />
  </div>
)
```

Add a route for it in [app/router.tsx](../frontend/src/app/router.tsx), inside the signed-in section next
to the dashboard:

```tsx
<Route path="tasks" element={<TasksPage />} />
```

Then add a link to `LINKS` in [app/layout/app-nav.tsx](../frontend/src/app/layout/app-nav.tsx), with an
icon from `lucide-react`. The sidebar and the mobile menu both read that list.

```tsx
import { LayoutDashboard, ListTodo } from 'lucide-react'

const LINKS = [
  { to: '/', label: 'Dashboard', icon: LayoutDashboard },
  { to: '/tasks', label: 'Tasks', icon: ListTodo },
]
```

### 13. Sample data

`make seed` creates fictional accounts. Give your feature fictional records too, so everyone on the team,
and staging, starts from the same data. Add a command in your app,
`tasks/management/commands/seed_tasks.py` (with an empty `__init__.py` in `management/` and in
`commands/`):

```python
from django.core.management.base import BaseCommand

from accounts.models import User
from tasks.models import Task


class Command(BaseCommand):
    help = 'Create fictional tasks for the seed accounts.'

    def handle(self, *args, **options):
        alex = User.objects.get(email='alex@example.com')

        for title in ['Book the minibus', 'Collect consent forms', 'Send the kit list']:
            Task.objects.get_or_create(owner=alex, title=title)
```

Then add it to the `seed` line in the `Makefile`, after the accounts:

```makefile
seed:
	docker compose exec backend uv run python manage.py seed
	docker compose exec backend uv run python manage.py seed_tasks
```

The brief's sample data, `example_inbound.csv`, is in the root of the repository. To load it, copy it into
your app, such as `tasks/sample_data.csv` (the backend container can only see `backend/`), and read it in
the command instead of typing the records out. The column names come from the file's first row:

```python
import csv
from pathlib import Path

SAMPLE_DATA = Path(__file__).resolve().parents[2] / 'sample_data.csv'


def handle(self, *args, **options):
    alex = User.objects.get(email='alex@example.com')

    with SAMPLE_DATA.open(newline='') as file:
        for row in csv.DictReader(file):
            Task.objects.get_or_create(owner=alex, title=row['title'])
```

## Checklist before you open a pull request

- [ ] The model has a migration, and it is committed.
- [ ] `get_queryset` only returns the signed-in user's rows (or, if the data is shared, you have decided
      who can change it and tested that).
- [ ] There are backend tests for each endpoint and for someone else's data.
- [ ] The list handles loading, error and empty states.
- [ ] `make test` passes.
