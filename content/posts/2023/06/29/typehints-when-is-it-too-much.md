---
title: "Typehints: When Is It Too Much?"
date: 2023-07-08T16:04:52-05:00
draft: false
categories: [thinking out loud]
---


As a python developer who learned on the job, I discovered typehints well into my career. I think of them as guide rails, or an extra layer of validation within your IDE and CICD. I do not lead with them when writing new code – typically I start with what abstractions best serve my goal, which interfaces are simplest, and then as a last flourish sprinkle typehints in.

Now, as a senior developer on a global team, I work with people of various technical backgrounds. Some have very specific knowledge like GIS and rasters, others cut their teeth on libraries like Pandas and use it everywhere. My favorites are those who mastered the standard library. However, some come into Python with a background in lower level languages like Java and that is where this post starts.

A junior developer was tasked with performing optional input validation on a class while not modifying the existing class. I envisioned the following, but knew there were several ways to skin this cat.

```python
class AuthAccount:  # ParentClass
    def __init__(**kwargs):
        self.a = kwargs['a']
        ...

class MachineryAuthAccount(AuthAccount):  #ChildClass
    def __init__(**kwargs):
        output = self.validate(**kwargs)
        super().__init__(**output)

    def validate(**kwargs):
        # the specific logic to be implemented, all the complexity to your hearts desire
        ...

```

The developer produced this:

```python
from __future__ import annotations

import dataclasses
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum, auto
from typing import (
    TYPE_CHECKING,
    AbstractSet,
    Any,
    Collection,
    Dict,
    Iterable,
    List,
    Literal,
    MutableMapping,
    Optional,
    Set,
    Tuple,
    TypeVar,
    Union,
    cast,
    final,
    overload,
)

import dateparser
from some_library.typing_utils.json import JSONObject
from typing_extensions import TypeAlias, TypeGuard

from application.v2.logic.auth_account import AuthAccount

if TYPE_CHECKING:
    from typing import SupportsIndex

    from _typeshed import SupportsKeysAndGetItem

__all__ = [
    "MachineryAuthAccount",
    "MachineryAuthAccountSettings",
    "MachineryAuthAccountSettingsRawT",
    "AutoFFTSettings",
    "StrDict",
    "StrSet",
    "AuthAccountUser",
    "AuthAccountUserList",
    "Status",
]

MachineryAuthAccountSettingsRawT: TypeAlias = JSONObject


class MachineryAuthAccount(AuthAccount):
    def __init__(
        self, *, settings: Union[MachineryAuthAccountSettings, MachineryAuthAccountSettingsRawT, None] = None, **kwargs
    ):
        if settings is None:
            settings = MachineryAuthAccountSettings()
        elif not isinstance(settings, MachineryAuthAccountSettings):
            settings = MachineryAuthAccountSettings.fromjson(settings)
        super().__init__(settings=settings, **kwargs)

    @property
    def settings(self) -> MachineryAuthAccountSettings:
        return cast(MachineryAuthAccountSettings, super().settings)

    @settings.setter
    def settings(self, settings: Union[MachineryAuthAccountSettings, MachineryAuthAccountSettingsRawT]):
        if not isinstance(settings, MachineryAuthAccountSettings):
            settings = MachineryAuthAccountSettings.fromjson(settings)
        return AuthAccount.settings.fset(self, settings)

    def export(self) -> JSONObject:
        exported = super().export()
        exported["settings"] = self.settings.tojson()
        return exported


class _DictProxy(MutableMapping[str, Any]):
    def __getitem__(self, key: str, /):
        try:
            result = getattr(self, key)
        except AttributeError as e:
            raise KeyError(key) from e
        if result is None:
            raise KeyError(key)
        return result

    def __setitem__(self, key: str, value: Any, /):
        try:
            return setattr(self, key, value)
        except AttributeError as e:
            raise KeyError(key) from e

    def __delitem__(self, key: str, /):
        if getattr(self, key, None) is None:
            raise KeyError(key)
        try:
            return delattr(self, key)
        except AttributeError as e:
            raise KeyError(key) from e

    def __contains__(self, key, /):
        return getattr(self, key, None) is not None

    @overload
    def get(self, key: str, default: None = None) -> Union[Any, None]:
        ...

    @overload
    def get(self, key: str, default: Any) -> Any:
        ...

    def get(self, key: str, default=None) -> Any:
        result = getattr(self, key, default)
        return default if result is None else result

    def __len__(self):
        v = vars(self)
        count = 0
        for value in v.values():
            if value is not None:
                count += 1
        return count

    def __iter__(self):
        for key, value in vars(self).items():
            if value is not None:
                yield key


@dataclass
class AutoFFTSettings(_DictProxy):
    is_active: bool = False

    @classmethod
    def fromjson(cls, data: JSONObject, /):
        return cls(**data)

    def tojson(self) -> JSONObject:
        return dataclasses.asdict(self)

    def __setattr__(self, name: str, value: Any, /):
        if name == "is_active":
            if not isinstance(value, bool):
                raise TypeError("is_active must be a bool")
        else:
            raise AttributeError(f"{self.__class__.__name__} has no field named {name!r}")
        return super().__setattr__(name, value)

    def __delattr__(self, name: str, /):
        if name == "is_active":
            return super().__setattr__(name, False)
        return super().__delattr__(name)

    @overload
    def __getitem__(self, key: Literal["is_active"], /) -> bool:
        ...

    @overload
    def __getitem__(self, key: str, /) -> Any:
        ...

    def __getitem__(self, key: str, /) -> Any:
        return super().__getitem__(key)

    @overload
    def get(self, key: Literal["is_active"], default: None = None) -> Optional[bool]:
        ...

    @overload
    def get(self, key: Literal["is_active"], default: _T) -> Union[bool, _T]:
        ...

    def get(self, key: str, default=None) -> Any:
        return super().get(key, default)


class StrDict(Dict[str, str]):
    def __init__(
        self,
        __map_or_iterable: Union[SupportsKeysAndGetItem[str, str], Iterable[Tuple[str, str]], None] = None,
        /,
        **kwargs: str,
    ):
        super().__init__()
        self.update(__map_or_iterable, **kwargs)

    def copy(self):
        return self.__class__(self)

    __copy__ = copy

    def __deepcopy__(self, memo=None):
        return self.copy()  # strings don't need deep copying

    def update(
        self,
        __map_or_iterable: Union[SupportsKeysAndGetItem[str, str], Iterable[Tuple[str, str]], None] = None,
        /,
        **kwargs: str,
    ):
        if __map_or_iterable is not None:
            if isinstance(__map_or_iterable, StrDict):
                super().update(__map_or_iterable)
            elif isinstance(__map_or_iterable, dict):
                for key, value in cast(Dict[str, str], __map_or_iterable).items():
                    self[key] = value
            elif hasattr(type(__map_or_iterable), "keys"):
                for key in __map_or_iterable.keys():  # type: ignore
                    self[key] = __map_or_iterable[key]  # type: ignore
            else:
                for key, value in __map_or_iterable:
                    self[key] = value
        if kwargs:
            for key, value in kwargs.items():
                if not isinstance(value, str):
                    raise TypeError("values of StrDict must be strings")
                super().__setitem__(key, value)

    @classmethod
    def fromkeys(cls, __iterable: Iterable[str], __value: str, /):
        if not isinstance(__value, str):
            raise TypeError("values of StrDict must be strings")
        result = cls()
        for key in __iterable:
            if not isinstance(key, str):
                raise TypeError("keys of StrDict must be strings")
            super().__setitem__(result, key, __value)
        return result

    def __setitem__(self, key: str, value: str, /):
        if not isinstance(key, str):
            raise TypeError("keys of StrDict must be strings")
        if not isinstance(value, str):
            raise TypeError("values of StrDict must be strings")
        return super().__setitem__(key, value)

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}({super().__repr__()})" if self else f"{self.__class__.__name__}()"


_T = TypeVar("_T")


class StrSet(Set[str]):
    @staticmethod
    def striter(s: Iterable[_T], /) -> Tuple[Iterable[_T], Collection[str] | None]:
        if not isinstance(s, Collection):
            s = list(s)
        if StrSet.isstrcollection(s):
            return s, s  # type: ignore
        return s, None

    @staticmethod
    def isstrcollection(s: Iterable[Any], /) -> TypeGuard[Collection[str]]:
        return isinstance(s, Collection) and all(isinstance(e, str) for e in s)

    @staticmethod
    def isstrset(s: Any, /) -> TypeGuard[AbstractSet[str]]:
        return isinstance(s, AbstractSet) and (isinstance(s, StrSet) or all(isinstance(e, str) for e in s))

    def __init__(self, __iterable: Optional[Iterable[str]] = None, /):
        super().__init__()
        if __iterable is not None:
            self.update(__iterable)

    def add(self, __element: str, /):
        if not isinstance(__element, str):
            raise TypeError("elements of StrSet must be strings")
        return super().add(__element)

    def copy(self):
        return self.__class__(self)

    __copy__ = copy

    def difference(self, *s: Iterable[Any]) -> StrSet:
        return StrSet(super().difference(*s))

    def intersection(self, *s: Iterable[Any]) -> StrSet:
        return StrSet(super().intersection(*s))

    @overload
    def symmetric_difference(self, __s: Iterable[str], /) -> StrSet:
        ...

    @overload
    def symmetric_difference(self, __s: Iterable[_T], /) -> Set[Union[str, _T]]:
        ...

    def symmetric_difference(self, __s, /) -> Union[StrSet, set]:
        __s, __s_str = StrSet.striter(__s)
        if __s_str is not None:
            return StrSet(super().symmetric_difference(__s_str))
        else:
            return super().symmetric_difference(__s)  # type: ignore

    def symmetric_difference_update(self, __s: Iterable[str], /):
        if not isinstance(__s, Collection):
            __s = list(__s)
        for e in __s:
            if not isinstance(e, str):
                raise TypeError("values of StrSet must be strings")
        super().symmetric_difference_update(__s)

    @overload
    def union(self, *s: Iterable[str]) -> StrSet:
        ...

    @overload
    def union(self, *s: Iterable[_T]) -> Set[Union[str, _T]]:
        ...

    def union(self, *s: Union[StrSet, Iterable[_T]]) -> Union[StrSet, Set[Union[str, _T]]]:
        s0 = []
        all_str = True
        for iterable in s:
            i0, i1 = StrSet.striter(iterable)
            if i1 is None:
                all_str = False
                s0.append(i0)
            else:
                s0.append(i1)
        if all_str:
            return StrSet(super().union(*s0))
        else:
            return super().union(*s0)

    def update(self, *s: Iterable[str]):
        for iterable in s:
            for e in iterable:
                self.add(e)

    def __and__(self, __value: AbstractSet[object], /) -> StrSet:
        if not isinstance(__value, AbstractSet):
            return NotImplemented
        return StrSet(super().__and__(__value))

    @overload
    def __or__(self, __value: AbstractSet[str], /) -> StrSet:
        ...

    @overload
    def __or__(self, __value: AbstractSet[_T], /) -> Set[Union[str, _T]]:
        ...

    def __or__(self, __value: Union[StrSet, AbstractSet[_T]], /) -> Union[StrSet, Set[Union[str, _T]]]:
        if StrSet.isstrset(__value):
            return StrSet(super().__or__(__value))
        else:
            return super().__or__(__value)

    def __ior__(self, __value: AbstractSet[str], /):
        if not isinstance(__value, AbstractSet):
            return NotImplemented
        self.update(__value)
        return self

    @overload
    def __xor__(self, __value: AbstractSet[str], /) -> StrSet:
        ...

    @overload
    def __xor__(self, __value: AbstractSet[_T], /) -> Set[Union[str, _T]]:
        ...

    def __xor__(self, __value: Union[StrSet, AbstractSet[_T]], /) -> Union[StrSet, Set[Union[str, _T]]]:
        if StrSet.isstrset(__value):
            return StrSet(super().__xor__(__value))
        else:
            return super().__xor__(__value)

    def __ixor__(self, __value: AbstractSet[str], /):
        if not isinstance(__value, AbstractSet):
            return NotImplemented
        self.symmetric_difference_update(__value)
        return self

    def __repr__(self) -> str:
        return (
            f"{self.__class__.__name__}({{{', '.join(map(repr, self))}}})" if self else f"{self.__class__.__name__}()"
        )


@dataclass
class AuthAccountUser(_DictProxy):
    email: str
    partners: Set[str] = field(default_factory=StrSet)
    devices: Set[str] = field(default_factory=StrSet)
    start_time: datetime = field(default_factory=datetime.now)

    @classmethod
    def fromjson(cls, data: JSONObject, /):
        args: Dict[str, Any] = data.copy()
        args["start_time"] = dateparser.parse(cast(str, args["start_time"]))
        if "devices" in args:
            args["devices"] = StrSet(args["devices"])
        if "partners" in args:
            args["partners"] = StrSet(args["partners"])
        return cls(**args)

    def tojson(self) -> JSONObject:
        data = dict(vars(self))
        data["start_time"] = self.start_time.isoformat()
        if self.partners:
            data["partners"] = list(self.partners)
        else:
            del data["partners"]
        if self.devices:
            data["devices"] = list(self.devices)
        else:
            del data["devices"]
        return data

    @final
    def export(self):
        return self.tojson()

    def __setattr__(self, name: str, value: Any, /):
        if name == "start_time":
            if not isinstance(value, datetime):
                raise TypeError("start_time must be a datetime")
        elif name == "email":
            if not isinstance(value, str):
                raise TypeError("email must be a string")
            if value.count("@") != 1 or value.startswith("@") or value.endswith("@"):
                raise ValueError(f"not a valid email: {value!r}")
        elif name in {"partners", "devices"}:
            if not isinstance(value, StrSet):
                if not isinstance(value, set) or not all(isinstance(elem, str) for elem in value):
                    raise TypeError(f"{name} must be a set of strings")
                value = StrSet(value)
        else:
            raise AttributeError(f"{self.__class__.__name__} has no field named {name!r}")
        return super().__setattr__(name, value)

    def __delattr__(self, name: str, /):
        if name in {"email", "start_time"}:
            raise AttributeError(f"cannot delete {name!r} attribute")
        elif name in {"partners", "devices"}:
            getattr(self, name).clear()
        else:
            return super().__delattr__(name)

    @overload
    def __getitem__(self, key: Literal["email"], /) -> str:
        ...

    @overload
    def __getitem__(self, key: Literal["devices", "partners"], /) -> Set[str]:
        ...

    @overload
    def __getitem__(self, key: Literal["start_time"], /) -> datetime:
        ...

    def __getitem__(self, key: str, /) -> Any:
        return super().__getitem__(key)

    @overload
    def get(self, key: Literal["email"], default: None = None) -> Optional[str]:
        ...

    @overload
    def get(self, key: Literal["email"], default: _T) -> Union[str, _T]:
        ...

    @overload
    def get(self, key: Literal["devices", "partners"], default: None = None) -> Optional[Set[str]]:
        ...

    @overload
    def get(self, key: Literal["devices", "partners"], default: _T) -> Union[Set[str], _T]:
        ...

    @overload
    def get(self, key: Literal["start_time"], default: None = None) -> Optional[datetime]:
        ...

    @overload
    def get(self, key: Literal["start_time"], default: _T) -> Union[datetime, _T]:
        ...

    def get(self, key: str, default=None) -> Any:
        return super().get(key, default)


class AuthAccountUserList(List[AuthAccountUser]):
    def __init__(self, __iterable: Optional[Iterable[AuthAccountUser]] = None, /):
        super().__init__()
        if __iterable is not None:
            self.extend(__iterable)

    def copy(self):
        return self.__class__(self)

    __copy__ = copy

    def __deepcopy__(self, memo=None):
        from copy import deepcopy

        return self.__class__(deepcopy(elem, memo) for elem in self)

    def append(self, __elem: AuthAccountUser, /):
        if not isinstance(__elem, AuthAccountUser):
            if isinstance(__elem, dict):
                __elem = AuthAccountUser.fromjson(__elem)
            else:
                raise TypeError("elements of AuthAccountUserList must be instances of AuthAccountUser")
        return super().append(__elem)

    def extend(self, __iterable: Iterable[AuthAccountUser], /):
        for elem in __iterable:
            self.append(elem)

    def insert(self, __index: SupportsIndex, __elem: AuthAccountUser, /):
        if not isinstance(__elem, AuthAccountUser):
            if isinstance(__elem, dict):
                __elem = AuthAccountUser.fromjson(__elem)
            else:
                raise TypeError("elements of AuthAccountUserList must be instances of AuthAccountUser")
        return super().insert(__index, __elem)

    @overload
    def __setitem__(self, __key: SupportsIndex, __value: AuthAccountUser, /):
        ...

    @overload
    def __setitem__(self, __key: slice, __value: Iterable[AuthAccountUser], /):
        ...

    def __setitem__(self, __key: Union[SupportsIndex, slice], __value, /):
        if isinstance(__key, slice):
            __value = list(__value)
            for i, elem in enumerate(__value):
                if not isinstance(elem, AuthAccountUser):
                    if isinstance(elem, dict):
                        __value[i] = AuthAccountUser.fromjson(elem)
                    else:
                        raise TypeError("elements of AuthAccountUserList must be instances of AuthAccountUser")
            return super().__setitem__(__key, __value)
        else:
            if not isinstance(__value, AuthAccountUser):
                if isinstance(__value, dict):
                    __value = AuthAccountUser.fromjson(__value)
                else:
                    raise TypeError("elements of AuthAccountUserList must be instances of AuthAccountUser")
            return super().__setitem__(__key, __value)

    @overload
    def __add__(self, __value: List[AuthAccountUser], /) -> AuthAccountUserList:
        ...

    @overload
    def __add__(self, __value: List[_T], /) -> List[Union[AuthAccountUser, _T]]:
        ...

    def __add__(self, __value, /) -> Union[AuthAccountUserList, list]:
        if (
            isinstance(__value, AuthAccountUserList)
            or isinstance(__value, list)
            and all(isinstance(e, AuthAccountUser) for e in __value)
        ):
            return AuthAccountUserList(super().__add__(__value))
        else:
            return super().__add__(__value)

    def __iadd__(self, __value: Iterable[AuthAccountUser], /):
        self.extend(__value)
        return self

    def __mul__(self, __value: SupportsIndex, /):
        result = super().__mul__(__value)
        return AuthAccountUserList(result) if result is not NotImplemented else NotImplemented

    __rmul__ = __mul__

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}([{', '.join(map(repr, self))}])" if self else f"{self.__class__.__name__}()"


class Status(Enum):
    UNKNOWN = auto()
    INACTIVE = auto()
    ACTIVE = auto()


@dataclass
class MachineryAuthAccountSettings(_DictProxy):
    start_date: datetime = field(default_factory=datetime.now)
    is_active: Optional[bool] = None
    auto_fft: Optional[AutoFFTSettings] = None
    etags: Optional[Dict[str, str]] = None
    organizations: Set[str] = field(default_factory=StrSet)
    with_encryption: bool = False
    status: Optional[Status] = None
    isReady: Optional[bool] = None
    devices: Set[str] = field(default_factory=StrSet)
    users: List[AuthAccountUser] = field(default_factory=AuthAccountUserList)

    @classmethod
    def fromjson(cls, data: JSONObject, /):
        args: Dict[str, Any] = data.copy()
        if "start_date" in args:
            args["start_date"] = dateparser.parse(cast(str, args["start_date"]))
        if "organizations" in args:
            args["organizations"] = StrSet(args["organizations"])
        if args.get("etags") is not None:
            args["etags"] = StrDict(args["etags"])
        if args.get("auto_fft") is not None:
            args["auto_fft"] = AutoFFTSettings.fromjson(args["auto_fft"])
        if "devices" in args:
            args["devices"] = StrSet(args["devices"])
        if "users" in args:
            if not isinstance(users := args["users"], list):
                raise TypeError("settings.users must be a list, if present")
            args["users"] = AuthAccountUserList(map(AuthAccountUser.fromjson, users))
        return cls(**args)

    def tojson(self) -> JSONObject:
        settings = dict(vars(self))
        if self.auto_fft is not None:
            settings["auto_fft"] = self.auto_fft.tojson()
        else:
            del settings["auto_fft"]
        settings["organizations"] = list(self.organizations)
        settings["start_date"] = self.start_date.isoformat()
        if self.is_active is None:
            del settings["is_active"]
        if self.etags is None:
            del settings["etags"]
        else:
            settings["etags"] = dict(self.etags)
        if self.status is None:
            del settings["status"]
        if self.isReady is None:
            del settings["isReady"]
        if self.devices:
            settings["devices"] = list(self.devices)
        else:
            del settings["devices"]
        if self.users:
            settings["users"] = [user.tojson() for user in self.users]
        else:
            del settings["users"]
        return settings

    @final
    def export(self):
        return self.tojson()

    def __setattr__(self, name: str, value: Any, /):
        if name == "start_date":
            if isinstance(value, str):
                value = dateparser.parse(value)
            elif not isinstance(value, datetime):
                raise TypeError(f"{name} must be a datetime")
        elif name in {"is_active", "isReady"}:
            if value is not None and not isinstance(value, bool):
                raise TypeError(f"{name} must be a boolean or None")
        elif name == "with_encryption":
            if not isinstance(value, bool):
                raise TypeError(f"{name} must be a boolean")
        elif name == "etags":
            if value is not None:
                if not isinstance(value, StrDict):
                    if not isinstance(value, dict) or not all(
                        isinstance(key, str) and isinstance(elem, str) for key, elem in value.items()
                    ):
                        raise TypeError("etags must be a dict[str, str] or None")
                    value = StrDict(value)
        elif name in {"organizations", "devices"}:
            if not isinstance(value, StrSet):
                if not isinstance(value, (set, list)) or not all(isinstance(org, str) for org in value):
                    raise TypeError("organizations must be a set of strings")
                value = StrSet(value)
        elif name == "auto_fft":
            if value is not None:
                if not isinstance(value, AutoFFTSettings):
                    if isinstance(value, dict):
                        value = AutoFFTSettings.fromjson(value)
                    else:
                        raise TypeError(f"auto_fft must be an instance of {AutoFFTSettings.__name__} or None")
        elif name == "status":
            if value is not None:
                if isinstance(value, str):
                    value = Status[value]
                elif not isinstance(value, Status):
                    raise TypeError("status must be one of 'INACTIVE', 'ACTIVE', 'UNKNOWN', or None")
        elif name == "users":
            if not isinstance(value, AuthAccountUserList):
                if not isinstance(value, list):
                    raise TypeError(f"users must be a list of {AuthAccountUser.__name__}")
                userlist = AuthAccountUserList()
                for elem in value:
                    if not isinstance(elem, AuthAccountUser):
                        if isinstance(elem, dict):
                            elem = AuthAccountUser.fromjson(elem)
                        else:
                            raise TypeError(f"users must be a list of {AuthAccountUser.__name__}")
                    userlist.append(elem)
        else:
            raise AttributeError(f"{self.__class__.__name__} has no field named {name!r}")
        return super().__setattr__(name, value)

    def __delattr__(self, name: str, /):
        if name == "start_date":
            raise AttributeError(f"cannot delete {name!r} attribute")
        elif name in {"is_active", "isReady", "etags", "auto_fft", "status"}:
            return super().__setattr__(name, None)
        elif name == "with_encryption":
            return super().__setattr__(name, False)
        elif name in {"organizations", "devices", "users"}:
            getattr(self, name).clear()
        else:
            return super().__delattr__(name)

    @overload
    def __getitem__(self, key: Literal["start_date"], /) -> datetime:
        ...

    @overload
    def __getitem__(self, key: Literal["with_encryption", "is_active", "isReady"], /) -> bool:
        ...

    @overload
    def __getitem__(self, key: Literal["auto_fft"], /) -> AutoFFTSettings:
        ...

    @overload
    def __getitem__(self, key: Literal["etags"], /) -> Dict[str, str]:
        ...

    @overload
    def __getitem__(self, key: Literal["devices", "organizations"], /) -> Set[str]:
        ...

    @overload
    def __getitem__(self, key: Literal["status"], /) -> Literal["INACTIVE", "ACTIVE", "UNKNOWN"]:
        ...

    @overload
    def __getitem__(self, key: Literal["users"], /) -> List[AuthAccountUser]:
        ...

    def __getitem__(self, key: str, /) -> Any:
        return super().__getitem__(key)

    @overload
    def get(self, key: Literal["start_date"], default: None = None) -> Optional[datetime]:
        ...

    @overload
    def get(self, key: Literal["start_date"], default: _T) -> Union[datetime, _T]:
        ...

    @overload
    def get(self, key: Literal["with_encryption", "is_active", "isReady"], default: None = None) -> Optional[bool]:
        ...

    @overload
    def get(self, key: Literal["with_encryption", "is_active", "isReady"], default: _T) -> Union[bool, _T]:
        ...

    @overload
    def get(self, key: Literal["auto_fft"], default: None = None) -> Optional[AutoFFTSettings]:
        ...

    @overload
    def get(self, key: Literal["auto_fft"], default: _T) -> Union[AutoFFTSettings, _T]:
        ...

    @overload
    def get(self, key: Literal["etags"], default: None = None) -> Optional[Dict[str, str]]:
        ...

    @overload
    def get(self, key: Literal["etags"], default: _T) -> Union[Dict[str, str], _T]:
        ...

    @overload
    def get(self, key: Literal["devices", "organizations"], default: None = None) -> Optional[Set[str]]:
        ...

    @overload
    def get(self, key: Literal["devices", "organizations"], default: _T) -> Union[Set[str], _T]:
        ...

    @overload
    def get(self, key: Literal["status"], default: None = None) -> Literal["INACTIVE", "ACTIVE", "UNKNOWN", None]:
        ...

    @overload
    def get(self, key: Literal["status"], default: _T) -> Union[Literal["INACTIVE", "ACTIVE", "UNKNOWN", None], _T]:
        ...

    @overload
    def get(self, key: Literal["users"], default: None = None) -> Optional[List[AuthAccountUser]]:
        ...

    @overload
    def get(self, key: Literal["users"], default: _T) -> Union[List[AuthAccountUser], _T]:
        ...

    def get(self, key: str, default=None) -> Any:
        return super().get(key, default)
```

Before I can fairly critique this, I need to explain a few more things:

- This codebase uses kwargs *way* too much, and we see it in the parent class here, called AuthAccounts.
- The data within this AuthAccount structure was determined over time, by various teams, with no technical leadership. Literally the settings object was *intended* to be a garbage-can of configuration values.

I say these caveats because, if you *were* going to strongly type this code, some of this would *have* to look like this. So let’s be clear about what we’re talking about here:

## When is it too much?

Readability is an often-championed yet relative dimension. I foresee some devs finding the above code readable, as it does lay it all out for you. But this developer chose to use a dataclass, a Python abstraction *intended* to be simple, then forced it to carry all this complexity, functional or not. Would a comment illustrating the structure of this dictionary suffice instead?

When I first looked at this 1k LOC PR, three classes stood out: StrSet, StrDict, and _DictProxy. These classes, with their glorious, manually-defined _xor_ dunder method overloads, only existed for typing. Alone they equated to 25% of the PR’s diff and they didn’t provide any functionality. Wherever the line is, surely we’ve crossed it?

I want to assign a task to my developers, and have them understand the relevant code in as little time as possible. Strongly typing this code hinders that. It’s like receiving a technical manual from Ikea, but the first 50 pages are acknowledgements. You can skip over them, but you have to know where they end, and in this metaphor there is no table of contents or headers guiding you. Code has a cost, even if it’s meant for typing, testing, comments, or actual functionality. It’s our job to make sure each line pulls its weight. That is the leading strength of Python: it has a high functionality to LOC ratio.

### Overloads.

I don’t have much to say on this topic, besides this: One of the first objects you master in Python is the dictionary, and with that comes a familiarity of the .get method. Does the following annotation help that familiarity? Where is the sweet spot between time saved with typehints and time spent understanding them?

```python
 @overload
    def get(self, key: Literal["start_date"], default: None = None) -> Optional[datetime]:
        ...

    @overload
    def get(self, key: Literal["start_date"], default: _T) -> Union[datetime, _T]:
        ...

    @overload
    def get(self, key: Literal["with_encryption", "is_active", "isReady"], default: None = None) -> Optional[bool]:
        ...

    @overload
    def get(self, key: Literal["with_encryption", "is_active", "isReady"], default: _T) -> Union[bool, _T]:
        ...

    @overload
    def get(self, key: Literal["auto_fft"], default: None = None) -> Optional[AutoFFTSettings]:
        ...

    @overload
    def get(self, key: Literal["auto_fft"], default: _T) -> Union[AutoFFTSettings, _T]:
        ...

    @overload
    def get(self, key: Literal["etags"], default: None = None) -> Optional[Dict[str, str]]:
        ...

    @overload
    def get(self, key: Literal["etags"], default: _T) -> Union[Dict[str, str], _T]:
        ...

    @overload
    def get(self, key: Literal["devices", "organizations"], default: None = None) -> Optional[Set[str]]:
        ...

    @overload
    def get(self, key: Literal["devices", "organizations"], default: _T) -> Union[Set[str], _T]:
        ...

    @overload
    def get(self, key: Literal["status"], default: None = None) -> Literal["INACTIVE", "ACTIVE", "UNKNOWN", None]:
        ...

    @overload
    def get(self, key: Literal["status"], default: _T) -> Union[Literal["INACTIVE", "ACTIVE", "UNKNOWN", None], _T]:
        ...

    @overload
    def get(self, key: Literal["users"], default: None = None) -> Optional[List[AuthAccountUser]]:
        ...

    @overload
    def get(self, key: Literal["users"], default: _T) -> Union[List[AuthAccountUser], _T]:
        ...

    def get(self, key: str, default=None) -> Any:
        return super().get(key, default)
```

## Hindsight
I left this experience with these thoughts:

that this code can be so abstruse indicates an underlying complexity predating the junior developer
are input validation and strongly typing code different things? For now, I can’t explain the difference, but I feel yes.
our team needs to define a standard level of typehinting, just as we do with formatting and linting
The junior developer and I came to an agreement for now to draw the line at custom classes and dunder methods to satisfy typing – that’s too much for me. With a few more changes to AC, we got the PR to below 500 LOC and merged. Maybe we’ll revisit at some point to consider libraries like pydantic. And no, the name is not lost on me.