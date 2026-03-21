# Business Processes

This document describes the key end-to-end business processes of the Dog Rescue Adoption platform.
Each process is shown as a sequence diagram so you can understand the flow of interactions **before** reading the API or Events contracts.

## 1. Rescue Registration

A rescue organisation signs up to the platform so it can start listing dogs.

```mermaid
sequenceDiagram
    actor Rescue
    participant API
    participant MessageBroker as Message Broker

    Rescue->>API: POST /rescues (name, email, phone, address)
    API-->>Rescue: 201 Created (rescueId, registeredAt)
    API--)MessageBroker: publish rescue.registered event
```

**Outcome:** The rescue receives a unique `rescueId` that it uses in all subsequent requests to add or manage its dogs.
A `rescue.registered` event is published so downstream systems (e.g. welcome emails, analytics) can react.

---

## 2. Dog Listing

A registered rescue adds a dog to the platform, making it discoverable by prospective adopters.

```mermaid
sequenceDiagram
    actor Rescue
    participant API
    participant MessageBroker as Message Broker

    Rescue->>API: POST /rescues/{rescueId}/dogs (name, breed, ageYears, sex)
    API-->>Rescue: 201 Created (dogId, status: available)
    API--)MessageBroker: publish dog.listed event
```

**Outcome:** The dog's status is set to `available` and it appears in cross-rescue browse results immediately.
A `dog.listed` event is published so search indexes and notification services stay up to date.

---

## 3. Adopter Browses and Finds a Dog

A prospective adopter uses the platform to search for a suitable dog across all rescues.

```mermaid
sequenceDiagram
    actor Adopter
    participant API

    Adopter->>API: GET /dogs?breed=Labrador&maxAgeYears=3&status=available
    API-->>Adopter: 200 OK (paginated list of matching dogs)
    Adopter->>API: GET /dogs/{dogId}
    API-->>Adopter: 200 OK (full dog details + rescue info)
```

**Outcome:** The adopter identifies one or more dogs they are interested in and is ready to submit an adoption request.

---

## 4. Adoption Request Submission

The adopter submits an adoption request for a specific dog.

```mermaid
sequenceDiagram
    actor Adopter
    participant API
    participant MessageBroker as Message Broker

    Adopter->>API: POST /dogs/{dogId}/adoptions (adopterName, adopterEmail, adopterPhone, message)
    API-->>Adopter: 201 Created (adoptionId, submittedAt)
    API--)MessageBroker: publish adoption.requested event
    Note right of MessageBroker: Dog status moves to 'reserved'
```

**Outcome:** The rescue is notified (via the `adoption.requested` event) and can review the request.
The dog's status transitions to `reserved`, preventing new adoption requests while the rescue decides.

---

## 5. Dog Update and Removal

A rescue can update a dog's details (e.g. correct breed, change status) or remove a dog that has left the platform.

```mermaid
sequenceDiagram
    actor Rescue
    participant API
    participant MessageBroker as Message Broker

    alt Update dog details or status
        Rescue->>API: PUT /dogs/{dogId} (updated fields)
        API-->>Rescue: 200 OK (updated dog)
        API--)MessageBroker: publish dog.updated event
    else Remove dog from platform
        Rescue->>API: DELETE /dogs/{dogId}
        API-->>Rescue: 204 No Content
        API--)MessageBroker: publish dog.removed event
    end
```

**Outcome:** Changes are reflected in browse results immediately and downstream consumers are notified via the appropriate domain event.

---

## Summary

| Process | Actor | Key API Call | Domain Event |
|---------|-------|--------------|--------------|
| Rescue Registration | Rescue | `POST /rescues` | `rescue.registered` |
| Dog Listing | Rescue | `POST /rescues/{rescueId}/dogs` | `dog.listed` |
| Browse Dogs | Adopter | `GET /dogs` | — |
| Adoption Request | Adopter | `POST /dogs/{dogId}/adoptions` | `adoption.requested` |
| Update Dog | Rescue | `PUT /dogs/{dogId}` | `dog.updated` |
| Remove Dog | Rescue | `DELETE /dogs/{dogId}` | `dog.removed` |
