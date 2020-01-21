# Project Structure

- **cmd/fabric-ca-server** contains the main for the fabric-ca-server command.
- **cmd/fabric-ca-client** contains the main for the fabric-ca-client command.
- lib contains most of the code. 
    - a)** server.go** contains the main Server object, which is configured by serverconfig.go. 
    - b) **client.go** contains the main Client object, which is configured by clientconfig.go.
- **util/csp.go** contains the Crypto Service Provider implementation.
- lib/dbutil contains database utility functions.
- lib/ldap contains LDAP client code.
- **lib/spi** contains Service Provider Interface code for the user registry.
- lib/tls contains TLS related code for server and client.
- util contains various utility functions.

Emphasized Files are what we mainly focus


# Key Code Reference

## Fabric CA Server

{% plantuml %}

'skinparam linetype ortho
rectangle FabricCA{
package main{
    class main {
        + main()
    }
    class servercmd {
        + newCommand(){init()}
        + init()
    }
    main --> servercmd: main{newCommand()}

}

package lib{
    class server{
        + Init(){init()}
        - init(){initDefaultCA()}
        - initDefaultCA()
    }
    class ca{
        + initCA()
    }
    server --> ca: initDefaultCA{ca.initCA}
}

servercmd -> server: init{lib.server.Init()}

package "lib.idemix" as idemix{
    class issuer{
        + Issuer
        + NewIssuer(){initDefaultCA}
        + Init(){initKeyMaterial()}
        - IssuerPublicKey()
        - RevocationPublicKey()
        - initKeyMaterial() //i.cfg.IssuerPublicKeyfile， i.cfg.IssuerSecretKeyfile
    }
    class issuercredential{
        + IssuerCredential interface{Load(), Store(), GetIssuerKey(), SetIssuerKey(), NewIssuerKey()}
        + NewIssuerCredential() >> IssuerCredential
    }
    issuer --> issuercredential:initKeyMaterial(){issuercredential.NewIssuerCredential}
    class idemixlib{
        + Lib interface // newIssuerKey(), newCredential ...
        + NewIssuerKey()
        + newCredential()
    }
    issuercredential --> idemixlib: IssuerCredential interface{... NewIssuerKey()} --> idemixlib.NewIssuerKey()
}

ca -> issuer: initCA(){lib.server.idemix.issuer.NewIssuer()}
}

rectangle "Fabric idemix" as Fabric{
    package f_idemix{
        class issuerkey{

            + Check()
            + NewIssuerKey()
            + sethash()
                }
        idemixlib -> issuerkey:idemixlib.NewIssuerKey(){issuerkey.NewIssuerKey()}
        class credential{
            + NewCredential()
                }
        idemixlib -> credential:idemixlib.NewCredential(){Fabric.credential.newCredential()}
    }
}

{% endplantuml %}

### cmd/fabric-ca-server/main.go
we start at file cmd/fabric-ca-server/main.go
```go
// RunMain is the fabric-ca server main
func RunMain(args []string) error {
	// Save the os.Args
	saveOsArgs := os.Args
	os.Args = args

	cmdName := ""
	if len(args) > 1 {
		cmdName = args[1]
	}
	scmd := NewCommand(cmdName, blockingStart) 

	// Execute the command
	err := scmd.Execute()

	// Restore original os.Args
	os.Args = saveOsArgs

	return err
}
```
At line42,  "NewCommand(cmdName, blockingStart)" leads us to cmd/servercmd.go

### cmd/servercmd.go

```go
// NewCommand returns new ServerCmd ready for running
func NewCommand(name string, blockingStart bool) *ServerCmd {
	s := &ServerCmd{
		name:          name,
		blockingStart: blockingStart,
		myViper:       viper.New(),
	}
	s.init()
	return s
}
```

In this fuction, we first construct a pointer of ServerCmd struct. 

```go
// ServerCmd encapsulates cobra command that provides command line interface
// for the Fabric CA server and the configuration used by the Fabric CA server
type ServerCmd struct {
	// name of the fabric-ca-server command (init, start, version)
	name string
	// rootCmd is the cobra command
	rootCmd *cobra.Command
	// My viper instance
	myViper *viper.Viper
	// blockingStart indicates whether to block after starting the server or not
	blockingStart bool
	// cfgFileName is the name of the configuration file
	cfgFileName string
	// homeDirectory is the location of the server's home directory
	homeDirectory string
	// serverCfg is the server's configuration
	cfg *lib.ServerConfig
}
```
Then run the init() funtion
In the initcmd, first defien the RunE function of this initCmd
```go
// line 98
	initCmd.RunE = func(cmd *cobra.Command, args []string) error {
		if len(args) > 0 {
			return errors.Errorf(extraArgsError, args, initCmd.UsageString())
		}
		err := s.getServer().Init(false)
		if err != nil {
			util.Fatal("Initialization failure: %s", err)
		}
		log.Info("Initialization was successful")
		return nil
	}
```

Next is to add this cmd to rootcmd, which is cobra cmd object(cmd that could run in terminal)

```go
s.rootCmd.AddCommand(initCmd)
```

Following the same precedure, we create a rtartCmd of s then define its RunE function and then add it to rootCmd

Last but not least, call s.registerFlags() function to registerFlags registers command flags with viper

Go back into init() function detail, at RunE defining part ,we found out 
```go 
err := s.getServer().Init(false) //line 102
```

s.getServer() return an lib.server object, here we go, the main core of fabric-ca-server

### lib/server




