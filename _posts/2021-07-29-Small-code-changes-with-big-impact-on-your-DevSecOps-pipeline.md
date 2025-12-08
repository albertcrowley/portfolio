---
layout: post
title: Small Code Changes with Big Impact on Your DevSecOps Pipeline
author: Al Crowley
---

# Small Code Changes with Big Impact on Your DevSecOps Pipeline

I want to talk about moving your project into a DevSecOps philosophy. I’m not suggesting you rewrite your entire stack overnight. Instead, I want to focus on small, often line-level code changes that pay dividends after just an iteration or two.

My favorite definition of DevOps isn't actually a formal definition; it’s a story I heard at a conference. The speaker described a project plagued by reliability issues. System administrators were getting paged at all hours—usually 3 a.m.—for site failures that almost always boiled down to application bugs.

The solution? The team decided the developers would take over the operational role. If the site went down at 3 a.m., the developer’s pager went off.

The outcome was immediate: the number of catastrophic production failures dropped like a stone. You could take the pessimistic view that developers just wanted to sleep through the night. But the more likely reality is that the dev team simply didn't understand the operational environment until they had to live in it. Having first-hand experience with what breaks production provided the insight needed to write better code.

That is the essence of DevSecOps: You built it, you secure it, and you run it.

## Testing: Put Value on Semantic Markup

Let’s start with the testing phase. One of the easiest wins you can get is using semantic tags. If you aren't familiar with the term, semantic tags (like <h1>, <nav>, or <figure>) convey meaning about the content, not just layout.

A generic <div> tells you nothing, but an <h1> is clearly a heading. While this is critical for accessibility—which you should be doing anyway—it also makes parsing and testing your application significantly easier.

Consider an XPath finder. If you use semantic tags, your test code becomes cleaner and more robust. If you use generic spans and divs, you force your tests into brittle, complex selectors.

```
!-- Good: Semantic tags allow clean XPath selectors -->
<h1>
  <nav>
    <label>
      <figure>
<!-- xpath: //h2//figure//label -->

<!-- Bad: Generic tags require brittle selectors -->
<div>
  <span>
    <p>
<!-- xpath: //div[contains(@class, "heading-medium")]//.... -->
```

Even better? Put unique IDs on any tag you might remotely be interested in. Picking out a "Next Page" button is much harder when there are two on the page, but trivial if you’ve given them unique IDs.

# Hard to maintain
xpath=//div[@id='main-content']//div[3]//button [contains(.,'next')]

# Easy to maintain
xpath=//button[@id = 'lower-next-page')


## Automate Everything (With a Little Effort)

"Automate everything" sounds daunting, but individual pieces are actually fairly easy to tackle if you use the right tools. I’m talking about schema changes, environment variables, configuration settings, and package requirements.

If you don’t automate it for your teammates, it probably won’t happen in their development environment. Usually, the first time you hear about a missing configuration is during the daily stand-up when someone complains that Feature X is throwing SQL errors.

I highly recommend Ansible for this. While you could write shell scripts to install packages or update configs, Ansible handles the error handling and edge cases for you. For example, using the blockinfile module allows you to insert a specific block of text into a configuration file. It’s smart enough not to duplicate the text if you run the script twice, and it uses markers to find and update that text in future iterations.

- name: "[#7253] Put the search fields and weights into the local.inc"
  blockinfile:
  path: /etc/gforge/local.inc
  marker: "// {mark} ANSIBLE MANAGED"
  insertbefore: "End of customizations"
  block: |
  $sys_fields_to_search = array(
  array("field_name"=>"group_name", "weight"=>2),
  array("field_name"=>"unix_group_name", "weight"=>1.375),
  array("field_name"=>"short_description", "weight"=>1.625),
  // ...
  );


## Modernize Database Management

In the past, I used plain SQL scripts for database updates. It worked, but it was messy. Today, I’m a big fan of migration tools like Sequelize or Liquibase.

These tools allow you to version-control your schema changes. They run updates in both forward and reverse directions, meaning you can safely migrate up to a new version or roll back if something breaks. On my current project, SRT, the migration utility runs every time the application starts, guaranteeing the database schema matches the code deployed.
```
let upSql = [
'CREATE TABLE solicitations ("id" SERIAL PRIMARY KEY, "solNum" varchar UNIQUE, "active" boolean default True, ...)',
'insert into solicitations ("solNum", active) (select distinct solicitation_number, True from notice)',
'alter table "Predictions" add column active boolean default TRUE'
];

let downSql = [
'DROP TABLE solicitations',
'alter table "Predictions" drop column active'
];

module.exports = migrationUtils.migrateUpDown(upSql, downSql)
```

## Logging: Log Everything, Error Judiciously

As I get deeper into DevOps, I find myself logging more and more data. I used to worry about disk space, but storage is cheap. On NITRC, we started logging every single email transaction because debugging delivery issues was such a nightmare. Even sending 20,000 messages a day, a month’s worth of logs only took up about 27 MB.

However, you need a strategy.

First, use structured logs. We use a library called SeasLog that outputs the date, level, session ID, and precise timestamp automatically. This saves you from writing boilerplate formatting code and makes the logs machine-readable.

// Old way: unstructured and manual
$bytes = file_put_contents($sys_debug_mail_log_file, $log, FILE_APPEND);

// New way: Structured, consistent logging
SeasLog::log(SEASLOG_INFO, "Wrote ${bytes} bytes to the mail log in ${sys_debug_mail_log_file}...");


Second, reserve the "Error" level. Only use the "Error" flag for something you effectively must take action on. It doesn't have to trigger a 3 a.m. hotfix, but it should be significant enough to warrant opening a ticket. This discipline is critical for effective monitoring.

When you have structured logs, you can use tools like jq (a command-line JSON processor) or a JSON log viewer to filter and search through the noise instantly.

Security and Feature Flags

Finally, let's talk about the "Sec" in DevSecOps.

A simple but powerful tool to add to your kit is Fail2Ban. It scans log files for patterns—like repeated failed login attempts or rapid-fire downloads—and then takes action, usually blocking the offending IP address via the firewall.

```
[apache-download-abuse]
enabled = true
filter = download-abuse-filter
action = iptables-multiport[name=download-abuse, port="80, 443"]
logpath = /var/log/httpd/download_abuse.log
maxretry = 5
bantime = 180
findtime = 50
```

Another technique that bridges operations and stability is Feature Flags. These let you turn specific features on or off without redeploying code.

We recently used this on NITRC when a new search engine library turned out to be unstable on our AWS servers. Instead of scrambling to build and deploy a hotfix while the site crashed intermittently, we simply flipped the feature flag to revert to the old search engine.

```
function printSearchForm() {
// If the feature is flagged on, use the new Solr search
if (getFeatureFlag('search-use_solr_group_search')) {
return $this->printSolrSearchForm();
}
// Otherwise fall back to the legacy search
}
```

We avoided an outage and bought ourselves time to fix the issue properly in the next sprint.

These changes—semantic tags, automation, migrations, structured logging, and safety valves like feature flags—might seem small in isolation. But collectively, they transform your pipeline from a series of manual headaches into a robust, secure, and efficient system.