+++
title = 'The Challenge of Evolving Schemas in Message Queues'
date = 2024-02-18T11:04:18+03:00

images = ['images/message_queuing.png']

tags = ['design', 'message-queuing']

categories = ['design']

noComment = false
+++

In systems built on message queues like RabbitMQ or Pub/Sub, schema evolution is inevitable. 
It allows you to adapt to changing needs and add new features, but it can also introduce challenges.
When updating schemas, ensuring a smooth transition is crucial to avoid disrupting existing processes and consumers. 
Let's explore a scenario where your schema undergoes significant changes:

Initial Payload (Version 1.0):

```json {linenos=inline}
{
  "eventType": "user_created",
  "version": "1.0",
  "timestamp": "2024-01-15T08:00:00Z",
  "data": {
    "id": "123456789",
    "username": "john_doe",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "age": 30,
    "gender": "male",
    "address": {
      "street": "123 Main St",
      "city": "Anytown",
      "state": "CA",
      "postal_code": "12345",
      "country": "USA"
    },
    "phone_numbers": [
      {
        "type": "home",
        "number": "123-456-7890"
      }
    ],
    "is_active": true,
    "registration_date": "2024-01-15T08:00:00Z"
  }
}
```
Then, suppose you introduce changes to this JSON structure as follows(Version 2.0):

```json {linenos=inline}
{
  "eventType": "user_created",
  "version": "2.0",
  "timestamp": "2024-01-15T08:00:00Z",
  "data": {
    "userId": "123456789",
    "profileDetails": {
      "username": "john_doe",
      "fullName": "John Doe",
      "email": "john.doe@example.com",
      "age": 30,
      "gender": "male"
    },
    "contact": {
      "address": {
        "street": "123 Main St",
        "city": "Anytown",
        "state": "CA",
        "postalCode": "12345",
        "country": "USA"
      },
      "phoneNumbers": [
        {
          "type": "home",
          "number": "123-456-7890"
        }
      ]
    },
    "isActive": true,
    "createdAt": "2024-01-15T08:00:00Z"
  }
}
```

Now, you need to migrate your system to handle these changes seamlessly.


## A Solution

1. **Create a New Topic:** Establish a new topic dedicated to publishing messages adhering to the updated schema versions. 
Producer publishes messages to the appropriate topic based on the schema used. This allows gradual migration and facilitates parallel operation of old and new consumers.

2. **Update the Producer:** Modify your producer to generate messages in the new schema format and direct these messages to the newly created topic.

3. **Implement the Adapter Pattern:** Develop an adapter component responsible for listening to messages in the new topic. The adapter, following the adapter pattern, translates messages from the new format to the old format and then transmits them to the original topic.

4. **Migration of Consumers:** Gradually migrate your consumers by redirecting them to subscribe to the new topic instead of the old one.

5. **Decommission the Adapter:** Once all consumers have successfully migrated, decommission the adapter, and remove the old topic from the system.

## Implementation Procedure

* User creation event version 1.

<img src="/images/step_1.jpg" alt="User event v1" title="User event v1">

<br>

* The adapter listens for messages in the new topic, translates them to the old format, and then forwards them to the old topic

<img src="/images/step_2.jpg" alt="User event v1" title="User event v1">

<br>

* In the next step, one consumer has migrated, with the other soon to follow.

<img src="/images/step_3.jpg" alt="User event v1" title="User event v1">

<br>

* In the final step, the adapter becomes obsolete, and the old topic is no longer needed.

<img src="/images/step_4.jpg" alt="User event v1" title="User event v1">

<br>

## Benefits of this approach

- **Backward Compatibility:** The Adapter ensures backward compatibility by acting as a bridge between the old and new schema formats, allowing existing Consumers to continue processing messages without disruption.

- **Flexibility:** Leveraging the adapter pattern provides flexibility in accommodating changes in schema versions or message formats, enabling incremental updates without impacting existing functionality.

- **Risk Mitigation:** The adapter pattern serves as a risk mitigation strategy by isolating the impact of schema changes, reducing the risk of disruptions or errors in message processing.

- **Clear Versioning:** Versioned topics and a schema registry provide clarity and transparency, making debugging and monitoring easier.

## Conclusion

Navigating schema evolution in message queue systems demands a strategic approach to ensure seamless transitions. By adopting the outlined solution, organizations can effectively address the challenges associated with schema updates while preserving operational continuity.

The proposed strategy acknowledges the inevitability of schema evolution and provides a structured approach to handle changes efficiently. By leveraging the Adapter Pattern, organizations can bridge the gap between old and new schema formats, facilitating backward compatibility and minimizing disruptions to existing processes.

Furthermore, the approach offers flexibility and risk mitigation, allowing for incremental updates and isolating the impact of schema changes. This ensures that system evolution occurs smoothly, with minimal risk to operational stability.