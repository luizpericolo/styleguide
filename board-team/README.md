
Those are the agreements we have for the projects within the board team. Feel free to copy, make suggestions, or even disagree!


# Endpoints and HTTP verbs

We believe that the endpoints should be vebose. We also enforce the use of the correct http verbs.

### GET
Should be used to retrieve a list of objects or an specific object.

```
- endpoint pattern: "<product>/<entity_pk>/<resource>/" or "<product>/<entity_pk>/<resource>/<resource_pk>/"
- eg: "board/7/meetings/" or "board/7/meetings/8/" if retrieving an specific object
```

### POST
Should be used to create a new object or execute a specific action

```
- endpoint pattern: "<product>/<entity_pk>/<resource>/" or "<product>/<entity_pk>/<resource>/<action>"
- eg: "board/7/meetings/" or "board/7/meetings/invite-attendees/"
```

### PUT
Used to update a particular object from a resource

```
- endpoint pattern: "<product>/<entity_pk>/<resource>/<resource_pk>/"
- eg: "board/7/meetings/8/"
```

### PATCH
Used to partially update a particular object from a resource

```
- endpoint pattern: "<product>/<entity_pk>/<resource>/<resource_pk>/"
- eg: "board/7/meetings/8/"
```

### DELETE
Used to destroy, even if virtually, an object from a resource

```
- endpoint pattern: "<product>/<entity_pk>/<resource>/<resource_pk>/"
- eg: "board/7/meetings/8/"
```


# Permissions

In board, the permission control is more granular than the rest of the app. There are 4 levelS of permissions:

* Corporation: Usually defined by the relationship of the user with the corporation. This permission set is declared at the endpoint
* Resource: Defined at the role level such as Company Admin, Board Admin, Board Member, and Meeting Attendee.
* Object: Some times only specific type of users can have access to objects. This is defined directly in the object level by selecting explicity the users with access to it.
* Action: There are some actions such as Sign Board Approvals that require the user to not just have access to the resource and object, but also access to a specific action. In this case, only board members can sign board approval, but the Board Admin can also have access to see the board approval.

### Permission names

We also believe that permissions should be verbose and easy to reuse without confusion. That's why we use constants to combine the roles and assign them to our endpoints.

```py
# bad
class IsBoardMemberOrAttendee:
    ...


# good
class BoardMember:
    ...


class MeetingAttendee:
    ...


CanViewBoardMeeting = Any(
    BoardMember(),
    MeetingAttendee(),
)
```


# Get rid of dead code

Unless it is a model without proper migrations, all code that is not being used should be killed. It also includes code in the frontend and libraries.


# Security


### Filters

Apart from the safety precautions we already need to have during our development time, we always filter our queries in our endpoints by the `corporation_pk` and `resource_pk`. Depending on the resource, we also include the `user_pk` on the filter. It avoids us to exposing protected data from bad intentioned users.

```py
# bad
BoardMeeting.objects.get(
    pk=kwargs['boardmeeting_pk']
)

# good
BoardMeeting.objects.get(
    pk=kwargs['boardmeeting_pk'],
    corporation_id=kwargs['corporation_pk']
)
```

### Viewset

An interesting approach to consider is to set the queryset of the DRF (Django Rest Framework) view as none and then implement the `get_queryset` method. It requires that the engineers always implement the `get_queryset` method, making sure that it will never expose to much data.

```py
# proposal
class BoardMeetingViewSet(mixins.ListModelMixin):
    model = BoardMeeting
    queryset = BoardMeeting.objects.none()

    def get_queryset(self):
        return self.model.objects.filter(
            corporation_id=self.kwargs['corporation_pk']
        )
```

# Single responsibility principle

We strongly believe on SRP. That means that we avoid to have fat models, specially when the models are doing way more than being simple models. That also means that we don't stick all the business logic on the models. We use serializers for validations, models for data integrity and retrieve data, and other classes to do the rest.

```py
# it works
class BoardMeeting(models.Model):
    ...

    def invite_attendees(self):
        ...

    def can_update(self):
        ...


# but this is better
class BoardMeeting(models.Model):
    ...

    def can_update(self):
        ...


class MeetingHost:

    def __init__(meeting):
        self.meeting = meeting
    
    def invite_attendees(self):
        ...
```

Yes, you might end up writing more code, but better and clearer code. Also it becomes easier to test and add behaviors.


# Model names

We prefer to use the model name and that's it. We have some legacy models that are not using this pattern and have the app name up front like `BoardMeeting`. Just meeting would be better. But that's fine for legacy.

```py
# bad
class BoardMeetingAttendee:
    ...

# good
class MeetingAttendee:
    ...
```

It makes easier to read not just in the application level but also in the database level. The code above will produce the tables `board_boardmeetingattendee` and `board_meetingattendee`. The second one seems to be closer to a real sentence.

# Tests

We aim 90% of test coverage, which means that we test most of the functions and classes we have. Keeping that in mind, we write code that is easy to test using the SRP concept not just for classes but also for functions and methods.

We also test different functions of the same resource separately. Take a look at the code below.

```py

class BoardMeetingViewSet:
    permission_classes = (CanViewBoardMeetings,)
    ...

    def publish(sef, request, *args, **kwargs):
        ...

    def send_invitations(self, request, *args, *kwargs):
        ...

```

In this code we would test the permission access in on test set, the publish method in another test set, and the send_invitations method in a different test set.

Integration tests are welcome but only for specific use cases. The ideal would be to have an integration test for a real case such as: John is the board admin for Meetly and needs to create a board meeting to invite the board members, Sarah, Rachael, and Zach.


# Data Migrations

We always consider data migrations in our sprints. Everything that touches the models has to check if data migrations are needed, it that's the case we write them as a proper migration. This is better to track all updates we make in our data base through scripts.

```py
def archive_all_meetings(apps, schema_editor):
    BoardMeeting = apps.get_model('board', 'BoardMeeting')

    meetings = BoardMeeting.objects.all()
    meetings.update(archived=date.today())


class Migration(migrations.Migration):

    dependencies = []
    operations = [
        migrations.RunPython(archive_all_meetings),
    ]
```


# Project structure

### Backend

We separate our models and views based on the domains of the product. Bellow you can see what an ideal structure would look like

```
board
  |---- models
  |       |--- meeting.py
  |       |--- approval.py
  |       |--- member.py
  |---- views
  |       |--- meeting.py
  |       |--- approval.py
  |       |--- approval.py
  |----  api
  |       |--- views
  |       |      |--- meeting.py
  |       |      |--- approval.py
  |       |      |--- approval.py
  |       |--- serializers
  |       |      |--- meeting.py
  |       |      |--- approval.py
  |       |      |--- approval.py
  |       |--- mobile
  |       |--- urls.py
  |---  urls.py
```

### Frontend

In the frontend our structure is a bit different. We have the structure based on the domain and then we break it down into components, containers, and helpers. It helps us find components of the application faster.
```
board
  |—— base
  |     |—— components
  |     |—— containers
  |     |—— helpers
  |     |—— app.jsx
  |—— approvals
  |     |—— components
  |     |—— containers
  |     |—— helpers
  |     |—— app.jsx
  |—— meetings
  |     |—— components
  |     |—— containers
  |     |—— helpers
  |     |—— app.jsx
  |—— members
  |     |—— components
  |     |—— containers
  |     |—— helpers
  |     |—— app.jsx

```

# Components name in the frontend

We try to standardize the names having first the component name and then its type.

```js
// bad
const ModalPDFItem;
const AgendaItemList;

// good
const PDFItemModal;
const AgendaItemListContainer;

// =================================

// bad
const DeleteMinutesModal;

// good
// We consider DeleteModal as the type of the object
const MinutesDeleteModal;
```

For codestyle we fallow most of this

https://github.com/carta/carta-web/pull/23504

Except for the section `Prefer DRF APIViews to DRF ViewSets`. We don't believe in any gain by doing this.


### Acknowledgments

* [Writing unit tests](https://github.com/carta/styleguide/tree/master/tests/unit)
