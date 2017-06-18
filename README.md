#         Run ASP.NET Core Application as a Service on Linux 

In this article, i'm going to review the steps you need to know in order to convert your DotNet Core application and deploy to a Linux server running Apache

##             Modify Dot Net Core project:

1.1 ) modify all project files: <project name>.csproj

Insert Linux runtime options into PropertyGroup section:

       <PropertyGroup>
			<AssemblyTitle>MicroServiceBePro.StaticTables</AssemblyTitle>
			<TargetFramework>netcoreapp1.1</TargetFramework>
			<AssemblyName>StaticLoader</AssemblyName>
			<PackageId>StaticLoader</PackageId>
			<NetStandardImplicitPackageVersion>1.6.1</NetStandardImplicitPackageVersion>
			<PackageTargetFallback>$(PackageTargetFallback);dnxcore50</PackageTargetFallback>
			<GenerateAssemblyConfigurationAttribute>false</GenerateAssemblyConfigurationAttribute>
			<GenerateAssemblyCompanyAttribute>false</GenerateAssemblyCompanyAttribute>
			<GenerateAssemblyProductAttribute>false</GenerateAssemblyProductAttribute>
		</PropertyGroup>
  
  insert new lines :
  
		  <RuntimeIdentifiers>win7-x64;win7-x86;ubuntu.16.04-x64;</RuntimeIdentifiers>
		  <SuppressDockerTargets>True</SuppressDockerTargets>
		  
  PropertyGroup section look like here:
	
	       <PropertyGroup>
			<AssemblyTitle>MicroServiceBePro.StaticTables</AssemblyTitle>
			<TargetFramework>netcoreapp1.1</TargetFramework>
			<AssemblyName>StaticLoader</AssemblyName>
			<PackageId>StaticLoader</PackageId>
			
			<PreserveCompilationContext>true</PreserveCompilationContext>
			<RuntimeIdentifiers>win7-x64;win7-x86;ubuntu.16.04-x64;</RuntimeIdentifiers>
			<SuppressDockerTargets>True</SuppressDockerTargets>
			

			<GenerateAssemblyConfigurationAttribute>false</GenerateAssemblyConfigurationAttribute>
			<GenerateAssemblyCompanyAttribute>false</GenerateAssemblyCompanyAttribute>
			<GenerateAssemblyProductAttribute>false</GenerateAssemblyProductAttribute>
		</PropertyGroup>
	
1.2) Run Visual Studio 2017 and open project:

Do operation

       - Clean Solution
	   - Rebuild Solution
	   
	   
1.2.3 - Tools -> Nuget Package Manager

       find and remove package from project: 
		       Microsoft.ApplicationInsights.AspNetCore.
       This package using for Microsoft Azure server. 
			
1.2.4 Remove from Startup.cs ( Startup class ) next code:

       if (env.IsDevelopment())
       {
          // This will push telemetry data through Application 
		  // Insights pipeline faster, allowing you to view results immediately.
          builder.AddApplicationInsightsSettings(developerMode: true);
       }		
			
1.2.5  Clean Solution, Rebuild All	
		
1.3 Run on the Command Prompt ( cmd ) and build linux executable feles
     		
		dotnet publish -c Release -r ubuntu.16.04-x64
		
		

##                 Modify Ubuntu Server.

2.1 Create application folder

         sudo mkdir /var/www/beprotb
		 
2.2 Upload Files using FTP Client application ( Like FileZilla )

		from:   bin/Release/netcoreapp1.1/ubuntu.16.04-x64/publish	
		to:	  /var/www/beprotb	 


2.3  Using chmod +x filename make the application executable.

			sudo chmod +x BeproTB


2.4  Launching Your App on Linux . Performance test your app before launch as service.

			sudo ./BeproTB
			
2.5 Show  running processes and kill then

Sample : 

		sudo ps aux | grep beprotb
		
			xxxx 18215 11.5 5.2 277244 108540 ? Sl 03:20 2:38 /var/www/beprotb/beprotb

		sudo kill 18215
			


###               Create linux service			
			
2.6  create service definition file:
	  
	       /lib/systemd/system/beprotb.service
		   
 			sudo nano /lib/systemd/system/beprotb.service
			
file context:
			
				[Unit]
					Description= BeproTB
					
					[Service]
					ExecStart=/var/www/beprotb/BeproTB
					Restart=always
					RestartSec=60
					SyslogIdentifier=beprotb
					Environment=ASPNETCORE_ENVIRONMENT=Production

					[Install]
				WantedBy=bepro.target
	  
2.7  make folder /etc/systemd/system/bepro.target.wants

2.8  make service as enabled :

			sudo  systemctl enable /lib/systemd/system/beprotb.service
			
2.9 start service :
				
			sudo service beprotb start

2.10 check service status:

			sudo service beprotb status	

you can use next command: 

			sudo service beprotb stop

			sudo service beprotb start

			sudo service beprotb restart

			sudo service beprotb status


###                  Create Apache Virtual Host 

2.11.1 Start by copying the file for the first domain:

	sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/beprotb.conf


2.11.2 Edit site definition on the folder  /etc/apache2/sites-available
			
		cd /etc/apache2/sites-available

		sudo nano beprotb.conf
					
file context:

		<VirtualHost *:80>
			ProxyPreserveHost On
			ProxyPass / http://http://127.0.0.1:5001/
			ProxyPassReverse / http://127.0.0.1:5001/
			ServerName beprotb.sgcombo.com
		</VirtualHost>

2.11.3 After you create virtual host file, you must enable them.

		sudo a2ensite beprotb.conf
					
2.11.4 Restart Apache service

		sudo service apache2 restart

You can use command:

		sudo service apache2 stop

		sudo service apache2 start

		sudo service apache2 restart

		sudo service apache2 status


## Configure a service to run at startup .

Those need an [Install] section with a WantedBy= that specifies the unit which that new service wants to become a dependency . Very commonly this is multi-user.target, which is roughly equivalent to start on runlevel in upstart; 

Modify service definition:

 
		[Unit]
			Description= UpStart Net Core App - BeproTB
			After=network.target network-online.target
			Wants=network-online.target

		[Service]
		   ExecStart=/var/www/beprotb2/BeproTB
		   WorkingDirectory=/var/www/beprotb2
		   StandardOutput=null
		   Restart=always
		   RestartSec= 120
		   Restart=on-failure
		   SystemlogIdentifier=beprotb
		   KillMode=control-group
		   Environment=ASPNETCORE_ENVIRONMENT=Production

		[Install]
		   WantedBy=multi-user.target
		   Alias=beprotb.service
		   
key After=network.target -  this service run only after network group		   
		   
After modify serice description you must reload systemd manager configuration

		sudo systemctl daemon-reload
		
Stop and start service:

			sudo service beprotb stop
			
Start the service and enable at start up.			
			
		sudo systemctl start beprotb.service
		sudo systemctl enable beprotb.service
			
Check if it's running::
		
   
		xxxx:~$ service beprotb status
		● beprotb.service - UpStart Net Core - BeproTB
		   Loaded: loaded (/lib/systemd/system/beprotb.service; enabled; vendor preset:
		   Active: active (running) since Sun 2017-06-18 10:47:07 UTC; 1h 32min ago
		 Main PID: 3876 (BeproTB)
			Tasks: 29
		   Memory: 9.9G
			  CPU: 15min 28.081s
		   CGroup: /system.slice/beprotb.service
				   └─3876 /var/www/beprotb2/BeproTB



