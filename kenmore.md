# Kenmore Appliance Support

**Year:** 2026
**Status:** Freelance · Live with full handover, shipped 2026-04-09
**Role:** Solo developer

---

## What it is

A self-hosted WordPress site for a US appliance repair business targeting two distinct intents: local search ("Kenmore refrigerator repair Chicago") and informational queries ("washer error code F21"). 116+ pages, 6 cities, 11 services, 10 DIY guides, all live, fully handed over to the client.

## Problem

The client is a regional appliance repair operator. Their old site was a 5-page WordPress install ranking on nothing. They wanted to capture both transactional search traffic ("repair in {city}") and long-tail informational traffic ("what does {error_code} mean on a {appliance}"). The content scope was 100+ pages, which would have taken months in the WP admin manually.

## Architecture

Custom theme + custom plugin, both hand-written PHP. No page builders, no Elementor, no third-party SEO plugins beyond Rank Math for technical SEO.

```
WordPress core
  │
  ├── theme: kenmore-repair (custom PHP)
  │   ├── 4 Custom Post Types: service, error_code, city, guide
  │   ├── 1 Taxonomy: appliance_type → drives /error-codes/{appliance}/ URL structure
  │   └── BreadcrumbList JSON-LD on every page
  │
  └── plugin: kenmore-content-importer (custom PHP)
      └── bulk-loads 116+ error_code entries, 6 cities, 11 services, 10 guides from CSV
```

The plugin is the multiplier. Hand-creating 116 error code pages would have meant 30+ hours in the WP admin clicking through forms. The importer reads from CSV, validates the schema, and bulk-creates the post type entries with the correct taxonomy assignments and meta fields. New rows in the CSV → re-run the import → new pages live in seconds.

The taxonomy `appliance_type` is what makes the error-code URL structure clean. Every error code is tagged with one of five appliance types; the theme rewrites URLs to `/error-codes/{appliance_type}/{error_code_slug}/` automatically. This gives both a flat URL (the error code page) and a nested category landing (`/error-codes/refrigerator/` lists all refrigerator error codes).

City pages bind structured location data: ZIP codes covered, suburb names, service hours, neighborhood-specific copy. JSON-LD on each city page exposes a `LocalBusiness` schema with the right `areaServed`.

## Stack

- **WordPress** core (self-hosted)
- **Theme:** custom PHP, no page builder
- **Plugin:** custom PHP, CSV-driven content importer
- **SEO:** Rank Math (technical), hand-written meta + schema for everything else
- **Design tokens:** Montserrat (Google Fonts), Kenmore blue `#008cc7`, inline SVG icons (no icon font dependency)
- **Local dev:** wp-now (npm) for fast iteration without Docker
- **Deploy:** ZIP upload via Hostinger File Manager, deliberately simple

## Verification

- 116+ error code pages live and indexed
- 6 city pages live with JSON-LD validating against schema.org
- 11 services + 10 guides live
- Full handover documentation: theme code, plugin code, CSV templates, deploy instructions
- Client can add new error codes by editing CSV and re-importing, no developer needed

## Notable decisions

1. **Custom plugin instead of CMS forms.** The 30-hours-of-clicking math is what made this an obvious automation. The plugin took 4 hours to write and saves all future content authoring forever. The client can scale to 500 error codes by adding rows to a CSV.

2. **Custom Post Types over hierarchical pages.** A WordPress page hierarchy with 116 nested children is technically possible and operationally a nightmare. CPTs let each error code, city, service, and guide be a discrete object with its own schema, taxonomy, and template, queryable independently.

3. **wp-now over Docker for local dev.** wp-now spins up a WordPress instance in a temp directory in 5 seconds, no MySQL container, no `docker-compose.yml`. For a single-author project this is dramatically faster to iterate on. Docker is the right answer for a team of 4; for a solo build, wp-now wins.

4. **Hostinger File Manager deploy.** No CI/CD pipeline. The client wanted a deploy they could understand and execute themselves if I'm unavailable. A ZIP upload is dumb, durable, and self-serve.

5. **Rank Math for technical SEO only; everything else hand-rolled.** Rank Math handles XML sitemap, redirects, basic meta. The actual page-level SEO (titles, descriptions, schema, internal linking) is theme-controlled. Plugin-controlled SEO is brittle because it depends on the plugin author's roadmap; theme-controlled SEO ships with the theme and survives a plugin migration.
