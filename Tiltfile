# Load the existing docker-compose.yml
docker_compose('./docker-compose.yml')

# Tell Tilt to build the gateway image from source (enables live updates)
docker_build(
  'channel-gateway',
  context='../gateway',
  dockerfile='../gateway/build/app/Dockerfile',
  ignore=['../gateway/.git', '../gateway/.venv', '../gateway/__pycache__'],
  live_update=[
    sync('../gateway/src', '/app/src'),
    sync('../gateway/channels', '/app/channels'),
    sync('../gateway/main.py', '/app/main.py'),
    restart_container(),
  ]
)

# ─── Organize resources in the Tilt UI ─────────────────────────────────────

# Chatwoot
dc_resource('cw-rails', labels=['chatwoot'],
  links=['http://localhost:8080'],
  resource_deps=['cw-postgres', 'cw-redis'])
dc_resource('cw-sidekiq', labels=['chatwoot'],
  resource_deps=['cw-postgres', 'cw-redis'])
dc_resource('cw-postgres', labels=['chatwoot'])
dc_resource('cw-redis', labels=['chatwoot'])

# Gateway
dc_resource('gw-init', labels=['gateway'],
  resource_deps=['cw-rails'])
dc_resource('gw-app', labels=['gateway'],
  links=['http://localhost:8000'],
  resource_deps=['gw-db', 'gw-init'])
dc_resource('gw-db', labels=['gateway'])
