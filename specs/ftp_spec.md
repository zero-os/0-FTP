# FTP server with 0-stor backend

![0-stor ftp server schema](https://gist.githubusercontent.com/chrisvdg/71f97a424b3fadf05c6d2d38720f177a/raw/f0dab929514497091da8dc151ccf5ad54511d0cc/schema.png)

# FTP server

With [fclairamb's ftpserver lib][fcliar] an FTP frontend for the [0-stor][zeroStor] can be made.

With the use of this lib, we can define a custom driver for the FTP server which will enable us to store the data onto the [0-stor][zeroStor].
The custom driver needs to implement following interfaces:

```go
// ServerDriver handles the authentication and ClientHandlingDriver selection
type ServerDriver interface {
	// Load some general settings around the server setup
	GetSettings() *Settings

	// WelcomeUser is called to send the very first welcome message
	WelcomeUser(cc ClientContext) (string, error)

	// UserLeft is called when the user disconnects, even if he never authenticated
	UserLeft(cc ClientContext)

	// AuthUser authenticates the user and selects an handling driver
	AuthUser(cc ClientContext, user, pass string) (ClientHandlingDriver, error)

	// GetCertificate returns a TLS Certificate to use
	// The certificate could frequently change if we use something like "let's encrypt"
	GetTLSConfig() (*tls.Config, error)
}

// ClientHandlingDriver handles the file system access logic
type ClientHandlingDriver interface {
	// ChangeDirectory changes the current working directory
	ChangeDirectory(cc ClientContext, directory string) error

	// MakeDirectory creates a directory
	MakeDirectory(cc ClientContext, directory string) error

	// ListFiles lists the files of a directory
	ListFiles(cc ClientContext) ([]os.FileInfo, error)

	// OpenFile opens a file in 3 possible modes: read, write, appending write (use appropriate flags)
	OpenFile(cc ClientContext, path string, flag int) (FileStream, error)

	// DeleteFile deletes a file or a directory
	DeleteFile(cc ClientContext, path string) error

	// GetFileInfo gets some info around a file or a directory
	GetFileInfo(cc ClientContext, path string) (os.FileInfo, error)

	// RenameFile renames a file or a directory
	RenameFile(cc ClientContext, from, to string) error

	// CanAllocate gives the approval to allocate some data
	CanAllocate(cc ClientContext, size int) (bool, error)
}
```

An example of an implemented driver can be found [here][sample_driver]

# Storage backend

The data of the files itself will be stored in [0-stor][zeroStor]

The metadata for the files will be stored in memory for quick access, but in case the FTP server would go down this data should be stored somewhere else so the files can be recovered.
To store the metadata Jan suggested the use of an SQL DB: [Cockroach][cockroach].  
An SQL database would allow for easy querying of the (relational) metadata.

[Cockroach][cockroach] offers the following features:
* horizontal scaling
* survives node failure with minimal latency disruption
* strongly-consistent ACID transactions

The database for the metadata could look something like this:

|file||||||||
|-|-|-|-|-|-|-|-|
|id(pk)|name|path_name(fk)|filemode|is_dir|mod_time|size|stor_key|

|path|
|-|
|name(pk)|

a quickstart guide for Cockroach can be found [here][cockroach_quickstart]  
A very crude example for this db schema with GORM and Cockroach can found [here][gorm_gist]

[zeroStor]:https://github.com/zero-os/0-stor
[fcliar]: https://github.com/fclairamb/ftpserver
[sample_driver]: https://github.com/fclairamb/ftpserver/blob/master/sample/sample_driver.go
[cockroach]: https://github.com/cockroachdb/cockroach
[cockroach_quickstart]: https://www.cockroachlabs.com/docs/stable/build-a-go-app-with-cockroachdb-gorm.html
[gorm_gist]: https://gist.github.com/chrisvdg/0858228b6843d46289232619bebe4bc4