# 1. System Architecture

## Overview

This document defines the high-level system architecture for **Qimmah (قمة)**, a mobile application for discovering officially approved hiking trails across Saudi Arabia.

The architecture follows a standard 3-tier model — Frontend, Backend, and Database — with integration to external third-party services for maps and image hosting.

## Architecture Diagram

```mermaid
flowchart TB
    Roles[User Roles]

    Roles --> actor1[Visitor]
    Roles --> actor2[Registered User]
    Roles --> actor3[Admin]

    actor1 --> FE
    actor2 --> FE
    actor3 --> FE

    FE["Frontend<br/>Flutter (Mobile App + Admin Web Dashboard)<br/>Browse · Search · Filter · Map · Trail Details · Save · Reviews · Admin"]

    FE -->|REST API| BE
    FE -->|Map Rendering| MAPS

    BE["Backend<br/>Flask (Python) + SQLAlchemy<br/>Auth JWT · Trails · Reviews · Saved Trails · Admin"]

    BE -->|SQL Queries| DB
    BE -->|Upload Photos| CLOUD

    DB["MySQL<br/>Users · Trails · Reviews · Saved Trails<br/>Trail Routes as GeoJSON"]

    CLOUD["Cloudinary<br/>Trail Photos · CDN"]

    MAPS["Google Maps SDK<br/>Trail Markers · GeoJSON Routes"]

    CLOUD -->|Image URLs| FE
