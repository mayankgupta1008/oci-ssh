# oci-ssh

SSH into your Oracle Cloud instance from a throwaway container.
The private key lives in a Docker volume, not on your laptop's disk.

## Prerequisites

- Docker installed and running
- pnpm installed (optional, but recommended)

## Setup

1. Copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and set:
   - Your instance's public IP in `OCI_HOST`
   - The name of your private key file in `OCI_KEY` (place this file in the project directory)
   ```bash
   # In .env file
   OCI_HOST=your-instance-public-ip
   OCI_KEY=your-private-key-file-name
   ```

3. Create the OCI CLI container and import your private key:
   ```bash
   # Using pnpm (recommended)
   pnpm create-vm
   ```

## Usage

After running `pnpm create-vm` for the initial setup, you can reconnect to your instance with:
```bash
docker compose run --rm ssh
```

## Additional Scripts

The project includes four main pnpm scripts for managing the OCI CLI:

- `pnpm create-vm` - Create the OCI CLI container, automatically import your private key, and connect to your instance
- `pnpm down` - Stop the Docker containers without removing volumes or images
- `pnpm nuke` - Stop containers, remove volumes, and delete all OCI-related Docker images and build cache
- `pnpm logs` - View and follow container logs