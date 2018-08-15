# Tools

This page describes the tools that are part of the framework. These tools are either intended to support the development process or to provide supporting functionality for applications built with SDACK.

## [ModificationSyncService](modificationsyncservice.md)

This is a simple REST service that provides a way to synchronize data modifications between multiple applications. The SDACK business framework directly supports this service so that it can be used to prevent concurrent modifications by separate applications. It comes with a command line tool that can be used to administrate a running sync service.

