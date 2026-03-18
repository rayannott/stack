---
title: "Validation & Serialization"
date: 2026-03-18
description: "Pydantic models for request and response, custom validators, Annotated field metadata, and serialization control."
weight: 30
draft: false
params:
  neso:
    show_toc: true
---

Using Pydantic to validate incoming data and shape outgoing responses.

<!--more-->

## Schemas: Input vs Output

A single model for both input and output leaks internal fields (password hashes, internal IDs) and accepts fields the client shouldn't control (`id`, `created_at`). Split into separate schemas.

```python
# src/schemas/users.py
from datetime import datetime

from pydantic import BaseModel, EmailStr


class UserBase(BaseModel):
    """Fields shared between input and output."""
    email: EmailStr
    display_name: str


class UserCreate(UserBase):
    """What the client sends when creating a user."""
    password: str


class UserUpdate(BaseModel):
    """Partial update --- all fields optional."""
    email: EmailStr | None = None
    display_name: str | None = None


class UserResponse(UserBase):
    """What the API returns. No password, has server-generated fields."""
    id: int
    created_at: datetime
```

Use them in route handlers:

```python
# src/routers/users.py
@router.post("/", status_code=201)
async def create_user(body: UserCreate, db: DbSession) -> UserResponse:
    user = await user_service.create(db, body)
    return UserResponse.model_validate(user)
```

> [!NOTE]
> `UserUpdate` does not inherit from `UserBase` because partial updates need every field to be optional. A shared base with required fields would defeat the purpose.

## Field Constraints with `Annotated`

Use `Annotated` with `Field`, `Query`, `Path`, `Body`, and `Header` to declare constraints once and reuse them across endpoints.

### Body fields (`Field`)

```python
from typing import Annotated

from pydantic import BaseModel, Field


class BookingCreate(BaseModel):
    room_id: Annotated[int, Field(ge=1, description="ID of the room to book")]
    guests: Annotated[int, Field(ge=1, le=20)]
    note: Annotated[str, Field(max_length=500)] = ""
```

### Query and path parameters

```python
from typing import Annotated

from fastapi import Path, Query


@router.get("/rooms")
async def list_rooms(
    floor: Annotated[int, Query(ge=1, le=50, description="Filter by floor")] = 1,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> list[RoomResponse]:
    ...


@router.get("/rooms/{room_id}")
async def get_room(
    room_id: Annotated[int, Path(ge=1)],
) -> RoomResponse:
    ...
```

### Headers

```python
from fastapi import Header


@router.get("/protected")
async def protected(
    x_api_key: Annotated[str, Header()],
) -> dict:
    ...
```

### Reusable constraints

Define constraint types once and use them everywhere:

```python
# src/schemas/common.py
from typing import Annotated

from pydantic import Field

PositiveInt = Annotated[int, Field(ge=1)]
ShortStr = Annotated[str, Field(min_length=1, max_length=255)]
```

```python
# src/schemas/bookings.py
from src.schemas.common import PositiveInt, ShortStr


class BookingCreate(BaseModel):
    room_id: PositiveInt
    title: ShortStr
```

## Custom Validators

### Single-field validation (`@field_validator`)

```python
from pydantic import BaseModel, field_validator


class SignupRequest(BaseModel):
    username: str
    email: str

    @field_validator("username")
    @classmethod
    def username_must_be_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("must be alphanumeric")
        return v.lower()  # normalize to lowercase
```

By default, `@field_validator` runs **after** Pydantic's built-in type coercion (mode `"after"`). Use `mode="before"` to transform raw input before type checking:

```python
@field_validator("tags", mode="before")
@classmethod
def split_tags(cls, v: str | list[str]) -> list[str]:
    if isinstance(v, str):
        return [t.strip() for t in v.split(",")]
    return v
```

### Cross-field validation (`@model_validator`)

```python
from pydantic import BaseModel, model_validator


class DateRange(BaseModel):
    start: datetime
    end: datetime

    @model_validator(mode="after")
    def end_must_be_after_start(self) -> "DateRange":
        if self.end <= self.start:
            raise ValueError("end must be after start")
        return self
```

`mode="after"` gives you a fully constructed model instance --- all fields are already validated and typed. Use `mode="before"` when you need to inspect or reshape raw input data before individual field validation runs.

### Reusable validators with `Annotated`

For validation logic you reuse across models, attach validators directly to the type:

```python
from typing import Annotated

from pydantic import AfterValidator


def must_be_lowercase(v: str) -> str:
    if v != v.lower():
        raise ValueError("must be lowercase")
    return v


LowercaseStr = Annotated[str, AfterValidator(must_be_lowercase)]
```

Now any field typed as `LowercaseStr` gets the validation automatically --- no decorator needed on every model.

## Serialization Control

### Return type vs `response_model`

Modern FastAPI (0.95+) infers the response schema from the return type annotation. Prefer this over the `response_model` parameter:

```python
# Preferred: return type annotation
@router.get("/users/{user_id}")
async def get_user(user_id: int, db: DbSession) -> UserResponse:
    ...

# Only use response_model when you need to filter fields from a
# model that has more fields than you want to expose
@router.get("/users/me", response_model=UserPublic)
async def get_current_user(db: DbSession) -> User:
    ...
```

### ORM integration (`from_attributes`)

To construct a Pydantic model directly from a SQLAlchemy (or any ORM) instance:

```python
from pydantic import BaseModel, ConfigDict


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str
    display_name: str
```

This lets you pass an ORM object directly:

```python
user = await db.get(User, user_id)
return UserResponse.model_validate(user)  # reads from user.id, user.email, etc.
```

### Computed fields

For fields derived from other data at serialization time:

```python
from pydantic import BaseModel, computed_field


class BookingResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    room_id: int
    guests: int
    price_per_guest: float

    @computed_field
    @property
    def total_price(self) -> float:
        return self.guests * self.price_per_guest
```

`total_price` appears in the JSON response and the OpenAPI schema, but is never stored --- it's computed on every serialization.
