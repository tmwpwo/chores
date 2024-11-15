based on this article: 
### [Request full access](https://developer.apple.com/documentation/eventkit/accessing_calendar_using_eventkit_and_eventkitui#4250785)

In iOS 17, an app with full access can create, edit, save, delete, and fetch all events on all the user’s calendars. Additionally, the app can display events using [`EKEventViewController`](https://developer.apple.com/documentation/eventkitui/ekeventviewcontroller) and allow the user to select another calendar using [`EKCalendarChooser`](https://developer.apple.com/documentation/eventkitui/ekcalendarchooser). Implement full access if your app needs to read and write data to Calendar. If your app only needs to write data directly to Calendar, implement write-only access instead. If your app only uses EventKit APIs to create and set up events, consider saving events to Calendar without prompting the user for authorization.

To implement full access in your app, follow these steps:

- Build your app with Xcode 15 and link against the iOS 17 SDK.
    
- Add the `NSCalendarsFullAccessUsageDescription` key to the `Info.plist` file of the target building your app.
    
- To request full access to events, use `requestFullAccessToEvents(completion:)` or `requestFullAccessToEvents()`.
    

Upon its first launch, the `MonthlyEvents` app registers for [`EKEventStoreChanged`](https://developer.apple.com/documentation/foundation/nsnotification/name/1507525-ekeventstorechanged) notifications to listen for any changes to the event store.

```
let center = NotificationCenter.default
let notifications = center.notifications(named: .EKEventStoreChanged).map({ (notification: Notification) in notification.name })


for await _ in notifications {
    guard await dataStore.isFullAccessAuthorized else { return }
    await self.fetchLatestEvents()
}
```

Then, the app checks whether it’s authorized to access the user’s calendar data.

```
let status = EKEventStore.authorizationStatus(for: .event)
```

If the authorization status is [`EKAuthorizationStatus.notDetermined`](https://developer.apple.com/documentation/eventkit/ekauthorizationstatus/notdetermined), the app uses an instance of [`EKEventStore`](https://developer.apple.com/documentation/eventkit/ekeventstore), `eventStore`, to prompt the user for full access.

```
return try await eventStore.requestFullAccessToEvents()
```

`MonthlyEvents` includes `NSCalendarsFullAccessUsageDescription` in its `Info.plist` file and uses its value when showing an alert. The alert prompts the user for full access to fetch events in all the user’s calendars and delete the ones the user selects in the app. If the user grants the request, the app receives a `.fullAccess` authorization status.

```
EKEventStore.authorizationStatus(for: .event) == .fullAccess
```

Then, the app fetches and displays all events occuring within a month in all the user’s calendars sorted by start date in ascending order.

```
let start = Date.now
let end = start.oneMonthOut
let predicate = eventStore.predicateForEvents(withStart: start, end: end, calendars: nil)
return eventStore.events(matching: predicate).sortedEventByAscendingDate()
```

If the user denies the request, the app does nothing. In subsequent launches, the app displays a message prompting the user to grant the app full access in Settings on their device.

Because the user authorized the app for full access, the user can additionally select and delete one or more events in `MonthlyEvents`. The app iterates through an array of events that the user chose to delete. It calls and sets the `commit` parameter of the [`remove(_:span:commit:)`](https://developer.apple.com/documentation/eventkit/ekeventstore/1507469-remove) function to `false` to batch the deletion of each event in the array.

```
try self.eventStore.remove(event, span: .thisEvent, commit: false)
```

Then, the app commits the changes once it’s done iterating through the array.

```
try eventStore.commit()
```

When you assign `true` to `commit` to immediately save or remove the event in your app, the event store automatically rolls back any changes if the commit operation fails. However, if you set `commit` to `false` and your app successfully removes some events and fails removing others, this can result in a later commit failing. Every subsequent commit fails until you roll back the changes. Call [`reset()`](https://developer.apple.com/documentation/eventkit/ekeventstore/1507345-reset) to manually roll back the changes.

```
eventStore.reset()
```



can you correct my code: 
//

//  choresApp.swift

//  chores

//

//  Created by Tomek Wester on 13/11/2024.

//

  

**import** SwiftUI

**import** EventKit

  

  

  

  

  

**@main**

**struct** chores: App{

    **var** body: **some** Scene{

        WindowGroup{

            ContentView()

        }

    }

}

  

**struct** ContentView: View {

    @StateObject **private** **var** dataManager = DataManager()

  

    **var** body: **some** View {

        VStack {

            Image(systemName: "globe")

                .imageScale(.large)

                .foregroundStyle(.tint)

            Text("Hello, world!")

            Button("Scrape & Send Data") {

                dataManager.requestAccess()

                dataManager.scrapeAndSendData()

            }

            .padding()

        }

    }

}

  

**class** DataManager: ObservableObject {

    **private** **let** eventStore = EKEventStore()

    **private** **let** endpointURL = URL(string: "http://localhost:8080/endpoint")!

  

    /// Request permissions for Calendar and Reminders

    **func** requestAccess() {

        eventStore.requestFullAccessToEvents() { granted, error **in**

            **if** !granted || error != **nil** {

                print("Calendar access denied or error occurred")

            }

        }

        eventStore.requestFullAccessToReminders(){ granted, error **in**

            **if** !granted || error != **nil** {

                print("Reminders access denied or error occurred")

            }

        }

    }

  

    /// Fetch events from Calendar and reminders, then send data to the endpoint

    **func** scrapeAndSendData() {

        **let** calendarEvents = fetchCalendarEvents()

        **let** reminders = fetchReminders()

        // Combine data into a dictionary

        **let** dataToSend: [String: **Any**] = [

            "calendarEvents": calendarEvents,

            "reminders": reminders

        ]

        // Convert to JSON and send to endpoint

        sendDataToEndpoint(data: dataToSend)

    }

  

    /// Fetch upcoming calendar events

    **private** **func** fetchCalendarEvents() -> [[String: **Any**]] {

        **var** eventsData = [[String: **Any**]]()

        // Define a start and end date (e.g., events for the next 7 days)

        **let** startDate = Date()

        **let** endDate = Calendar.current.date(byAdding: .day, value: 7, to: startDate)!

  

        **let** calendars = eventStore.calendars(for: .event)

        **let** predicate = eventStore.predicateForEvents(withStart: startDate, end: endDate, calendars: calendars)

        **let** events = eventStore.events(matching: predicate)

  

        **for** event **in** events {

            eventsData.append([

                "title": event.title ?? "No Title",

                "startDate": event.startDate ?? "startDate",

                "endDate": event.endDate ?? "endDate",

                "location": event.location ?? "No Location"

            ])

        }

  

        **return** eventsData

    }

  

    /// Fetch reminders

    **private** **func** fetchReminders() -> [[String: **Any**]] {

        **var** remindersData = [[String: **Any**]]()

        **let** semaphore = DispatchSemaphore(value: 0)

        **let** predicate = eventStore.predicateForReminders(in: **nil**)

        eventStore.fetchReminders(matching: predicate) { reminders **in**

            **if** **let** reminders = reminders {

                **for** reminder **in** reminders {

                    remindersData.append([

                        "title": reminder.title ?? "No Title",

                        "dueDate": reminder.dueDateComponents?.date ?? Date(),

                        "isCompleted": reminder.isCompleted

                    ])

                }

            }

            semaphore.signal()

        }

        _ = semaphore.wait(timeout: .distantFuture)

        **return** remindersData

    }

  

    /// Send data to the specified endpoint

    **private** **func** sendDataToEndpoint(data: [String: **Any**]) {

        **do** {

            **let** jsonData = **try** JSONSerialization.data(withJSONObject: data, options: .prettyPrinted)

            **var** request = URLRequest(url: endpointURL)

            request.httpMethod = "POST"

            request.setValue("application/json", forHTTPHeaderField: "Content-Type")

            request.httpBody = jsonData

  

            **let** task = URLSession.shared.dataTask(with: request) { data, response, error **in**

                **if** **let** error = error {

                    print("Error sending data: \(error)")

                    **return**

                }

                **if** **let** response = response **as**? HTTPURLResponse {

                    print("Server responded with status code: \(response.statusCode)")

                }

            }

            task.resume()

        } **catch** {

            print("Failed to serialize JSON: \(error)")

        }

    }

}