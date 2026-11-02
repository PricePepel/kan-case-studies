# Kenmore Appliance Support

**Year:** 2026
**Status:** Freelance · Shipped 2026-04-09
**Role:** Solo developer

---

## What it is

A self-hosted WordPress site for a US appliance repair business, targeting two kinds of search: local ("Kenmore refrigerator repair Chicago") and informational ("washer error code F21"). It has 116+ error code pages across five appliance lines, 6 city pages, 11 services, and 10 DIY guides.

## Problem

The site needed to serve both people looking for a repair in their city and people looking up what an error code means. At 100+ pages, entering the content by hand in the WordPress admin would have been slow and error-prone.

## Architecture

A custom theme and a custom plugin, both hand-written PHP.

```
WordPress core
  │
  ├── theme: kenmore-repair (custom PHP)
  │   ├── 4 Custom Post Types: service, error_code, city, guide
  │   ├── 1 Taxonomy: appliance_type → drives /error-codes/{appliance}/ URLs
  │   └── BreadcrumbList JSON-LD
  │
  └── plugin: kenmore-content-importer (custom PHP)
      └── import scripts for error codes, cities, guides, and static pages
```

The plugin is the multiplier. Its import scripts create the post type entries in bulk, with the right taxonomy assignments and fields, instead of someone filling in forms one page at a time.

The `appliance_type` taxonomy keeps the error code URLs clean. Each code is tagged with an appliance type, which gives both a page per code and a category landing page such as `/error-codes/refrigerator/`.

## Stack

- **WordPress** core (self-hosted)
- **Theme:** custom PHP, no page builder
- **Plugin:** custom PHP content importer
- **Design:** Montserrat, Kenmore blue `#008cc7`, inline SVG icons
- **Local dev:** wp-now
- **Deploy:** ZIP upload via Hostinger File Manager

## Notable decisions

1. **A custom import plugin instead of CMS forms.** At this page count, scripted imports were the only sensible way to keep the content consistent.

2. **Custom Post Types over hierarchical pages.** Each error code, city, service, and guide is its own object with its own template and taxonomy, and can be queried independently. A page hierarchy with 116 nested children would be much harder to manage.

3. **wp-now over Docker for local dev.** It starts a WordPress instance without a MySQL container or a compose file, which suits a single-author project.

4. **A ZIP deploy.** There's no CI/CD pipeline. A ZIP upload through the host's file manager is simple and easy to repeat.
