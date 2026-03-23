# Project: Modernization Analysis

## Tech stack
- Frontend: JSP / legacy MVC
- Backend: Java Spring + Oracle PLSQL
- Messaging: Apache Kafka
- APIs: Internal REST (JAX-RS)
- DAO layer: MyBatis

## Entry points
- Main JSPs: /src/main/webapp/WEB-INF/
- Services:  /src/main/java/com/acme/service/
- DAOs:      /src/main/java/com/acme/dao/
- PLSQL:     /db/procedures/
- Kafka:     /src/main/java/com/acme/kafka/

## Output directory
All chain artefacts → modernization-output/{slug}/

## Skills available
- /analyze-entry  — scan a legacy entry point (JSP, service, DAO, PLSQL, REST, Kafka)
- /gen-funcspec   — generate functional spec from analysis
- /gen-jira       — generate JIRA modernization stories
- /modernize      — run all three steps end-to-end
