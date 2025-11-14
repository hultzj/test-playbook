# Ansible Playbooks for AAP

A collection of simple Ansible playbooks designed for Ansible Automation Platform (AAP).

## Description

This repository contains multiple playbooks that demonstrate basic Ansible functionality. Each playbook can be configured as a separate Job Template in AAP.

## Requirements

- Ansible 2.9 or higher
- Ansible Automation Platform (AAP) 2.x or higher (for AAP usage)

## Files

- `hello-world.yml` - Simple greeting playbook
- `goodbye-world.yml` - Farewell message playbook
- `datetime.yml` - Displays current date/time information
- `inventory.yml` - Sample inventory file (localhost)

## Playbooks

### 1. Hello World (`hello-world.yml`)
Displays greeting messages and welcomes users.

**Usage:**
```bash
ansible-playbook -i inventory.yml hello-world.yml
```

### 2. Goodbye World (`goodbye-world.yml`)
Displays farewell messages.

**Usage:**
```bash
ansible-playbook -i inventory.yml goodbye-world.yml
```

### 3. DateTime (`datetime.yml`)
Gathers and displays comprehensive date/time information including ISO format, timezone, epoch time, and day of week.

**Usage:**
```bash
ansible-playbook -i inventory.yml datetime.yml
```

## Running in Ansible Automation Platform (AAP)

### Step 1: Create a Project

1. Navigate to **Resources → Projects**
2. Click **"Add"**
3. Configure:
   - **Name:** "Demo Playbooks"
   - **Source Control Type:** Git
   - **Source Control URL:** [Your repository URL]
   - **Source Control Branch/Tag/Commit:** main (or your branch)
4. Click **"Save"**

### Step 2: Create an Inventory

1. Navigate to **Resources → Inventories**
2. Either use an existing inventory or create a new one
3. Add hosts as needed (or import the included `inventory.yml`)

### Step 3: Create Job Templates

Create a separate Job Template for each playbook:

#### Hello World Job Template
- Navigate to **Resources → Templates → Add → Job Template**
- **Name:** "Hello World"
- **Job Type:** Run
- **Inventory:** [Select your inventory]
- **Project:** "Demo Playbooks"
- **Playbook:** `hello-world.yml`
- **Credentials:** [Add if needed]
- Click **"Save"**

#### Goodbye World Job Template
- **Name:** "Goodbye World"
- **Playbook:** `goodbye-world.yml`
- (Other settings same as above)

#### DateTime Job Template
- **Name:** "DateTime Info"
- **Playbook:** `datetime.yml`
- (Other settings same as above)

### Step 4: Launch Jobs

- Navigate to the Job Template you want to run
- Click the **rocket icon** 🚀 to launch
- View the output in the job details page

## Expected Output

### Hello World
- Greeting messages
- Welcome message
- Success confirmation

### Goodbye World
- Farewell messages
- See you later message
- Success confirmation

### DateTime
- ISO 8601 timestamp
- Human-readable date and time
- Timezone information
- Unix epoch time
- Day of week (number and name)

## Customization

You can modify any playbook to:
- Target different hosts by changing the `hosts` parameter
- Enable/disable fact gathering with `gather_facts`
- Add additional tasks as needed
- Change messages to suit your needs

## License

MIT

## Author

Generated for AAP Hello World demonstration

