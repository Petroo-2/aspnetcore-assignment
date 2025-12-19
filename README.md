<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>QueueManagementSystem</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <!-- Required for custom PostgreSQL data access -->
    <PackageReference Include="Npgsql" Version="6.0.7" /> 
    
    <!-- Required for secure password hashing -->
    <PackageReference Include="BCrypt.Net-Core" Version="1.6.0" />

    <!-- Assume FastReport.Net package reference would go here -->
    <!-- <PackageReference Include="FastReport.Net" Version="2023.2.14" /> -->
    
  </ItemGroup>

</Project>