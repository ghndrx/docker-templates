# Security Templates

Hardened Docker configurations following industry best practices.

## Quick Start

```bash
# Copy the hardened template
cp docker-compose.hardened.yml ../my-project/docker-compose.yml

# Validate configuration
docker compose config --quiet

# Test with your image
docker compose up -d
```

## Security Checklist

| Control | Risk | Implementation |
|---------|------|----------------|
| Non-root user | **Critical** | `user: "1000:1000"` |
| Read-only filesystem | High | `read_only: true` + tmpfs |
| Drop capabilities | High | `cap_drop: [ALL]` |
| No privilege escalation | High | `no-new-privileges:true` |
| Resource limits | Medium | `deploy.resources.limits` |
| Network isolation | Medium | Internal networks |
| Log rotation | Medium | `logging.options` |

## Common Capability Requirements

| Application Type | Capabilities Needed |
|-----------------|---------------------|
| Web server (port 80/443) | `NET_BIND_SERVICE` |
| Ping/network diagnostics | `NET_RAW` |
| File ownership changes | `CHOWN` |
| Process management | `SETUID`, `SETGID` |

## Security Testing

```bash
# Verify non-root
docker compose exec app whoami
# Should NOT return 'root'

# Test read-only filesystem
docker compose exec app touch /test
# Should fail with 'Read-only file system'

# Check capabilities
docker compose exec app capsh --print
# Should show minimal capabilities

# Audit with Docker Bench
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker/docker-bench-security
```

## References

- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Docker Security Documentation](https://docs.docker.com/engine/security/)
