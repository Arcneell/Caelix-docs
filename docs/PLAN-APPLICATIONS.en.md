# Plan: Applications, Domains & Certificates

## Context

Caelix is a Docker orchestrator (Bash engine + FastAPI backend + Vue 3 frontend). Currently, containers are managed individually. The goal is to make Caelix accessible to everyday users by adding:

1. **Applications**: group containers into named stacks (e.g., WordPress = nginx + mysql + phpmyadmin)
2. **Enhanced assistant**: guided templates (LAMP, LEMP, WordPress, etc.) with complementary service suggestions
3. **Domains**: virtual hosts, reverse proxy mapping, multiple domain names per app
4. **Certificates**: custom upload or auto Let's Encrypt/certbot

New "Applications" category at the top of the sidebar, deployed by default.

---

## Progress

- [x] Phase 1: Backend stacks (config.py, stack_templates.py, app_stacks.py, main.py) -- DONE
- [x] Phase 2: Frontend applications (sidebar, router, views, StackAssistant, i18n) -- DONE
- [x] Phase 3: Domains (backend domains.py + DomainsView.vue) -- DONE
- [x] Phase 4: Certificates (backend certificates.py + CertificatesView.vue) -- DONE
- [x] Phase 5: TLS proxy (proxy.sh + apply integration) -- DONE

### Phase 1 Detail
- [x] 1.1 config.py: new path constants
- [x] 1.2 core/stack_templates.py: built-in templates
- [x] 1.3 routers/app_stacks.py: full CRUD
- [x] 1.4 main.py: register router

### Phase 2 Detail
- [x] 2.1 i18n (en.ts + fr.ts)
- [x] 2.2 router/index.ts: new routes
- [x] 2.3 App.vue: sidebar Applications group
- [x] 2.4 AppStacksView.vue: stacks list
- [x] 2.5 AppStackDetailView.vue: stack detail
- [x] 2.6 StackTemplateSelector.vue: template grid
- [x] 2.7 StackAssistant.vue: 5-step wizard
- [x] 2.8 StackAssistantView.vue: assistant page
- [x] 2.9 DomainsView.vue (placeholder)
- [x] 2.10 CertificatesView.vue (placeholder)

### Phase 3 Detail
- [x] 3.1 routers/domains.py
- [x] 3.2 main.py: register domains router
- [x] 3.3 DomainsView.vue: full implementation
- [x] 3.4 DomainForm.vue

### Phase 4 Detail
- [x] 4.1 routers/certificates.py
- [x] 4.2 main.py: register certificates router
- [x] 4.3 CertificatesView.vue: full implementation
- [x] 4.4 CertificateUpload.vue

### Phase 5 Detail
- [x] 5.1 proxy.sh: TLS listener (OPENSSL-LISTEN socat + CLI args + env vars)
- [x] 5.2 autoscale.sh: pass TLS keys from manifest [proxy] to global proxy
- [x] 5.3 /api/domains/apply: auto-configure TLS in manifest when SSL domains exist
