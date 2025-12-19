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





using System;
using System.Collections.Generic;

namespace QueueManagementSystem.Models
{
    // --- Configuration Models ---

    public class Service
    {
        public int ServiceId { get; set; }
        public string Name { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public char TicketPrefix { get; set; }
        public bool IsActive { get; set; }
    }

    public class ServicePoint
    {
        public int PointId { get; set; }
        public string Name { get; set; } = string.Empty;
        public bool IsActive { get; set; }
    }

    public class ServiceProvider
    {
        public int ProviderId { get; set; }
        public string Username { get; set; } = string.Empty;
        public string PasswordHash { get; set; } = string.Empty; 
        public string FullName { get; set; } = string.Empty;
        public int? AssignedPointId { get; set; } 
        public bool IsAdmin { get; set; }
    }

    public class PointServiceMapping
    {
        public int MappingId { get; set; }
        public int PointId { get; set; }
        public int ServiceId { get; set; }
    }

    // --- Core Queue Model ---

    public class QueueTicket
    {
        public long TicketId { get; set; }
        public string TicketNumber { get; set; } = string.Empty;
        public int ServiceId { get; set; }
        
        // Status: 'Waiting', 'Called', 'NoShow', 'Finished', 'Transferred'
        public string Status { get; set; } = "Waiting"; 

        // Timestamps (for reporting and service flow)
        public DateTime IssueTime { get; set; }
        public DateTime? CallTime { get; set; }
        public DateTime? FinishTime { get; set; }

        // Associated personnel/location
        public int? ServicePointId { get; set; }
        public int? ServiceProviderId { get; set; }

        // Transfer tracking
        public int? TransferFromPointId { get; set; }
        
        // --- Display/View Properties (Not mapped directly to DB columns) ---
        public string ServiceName { get; set; } = string.Empty;
        public string ServicePointName { get; set; } = string.Empty;
        public string ProviderName { get; set; } = string.Empty;
        
        // Calculated Property for Analytics
        public TimeSpan WaitingTime => (CallTime ?? DateTime.Now) - IssueTime;
        public TimeSpan ServiceTime => (FinishTime ?? DateTime.Now) - (CallTime ?? DateTime.Now);
    }
    
    // --- View Model for Displaying Ticket/Point Info (Waiting Page) ---
    public class CurrentCallViewModel
    {
        public string TicketNumber { get; set; } = "---";
        public string ServicePointName { get; set; } = "---";
    }

    // --- View Model for Analytics Report ---
    public class AnalyticsReportData
    {
        public string Name { get; set; } = string.Empty; // Service Point Name or Provider Name
        public int CustomersServed { get; set; }
        public TimeSpan AverageWaitingTime { get; set; }
        public TimeSpan AverageServiceTime { get; set; }
        public string TimePeriod { get; set; } = "All Time";
    }
}





using Npgsql;
using QueueManagementSystem.Models;
using System.Collections.Generic;
using System.Threading.Tasks;
using System;
using BCrypt.Net;

namespace QueueManagementSystem.DataAccess
{
    // Implementation of all custom CRUD and operational methods required by the assignment
    public class NpgsqlDataAccess
    {
        private readonly string _connectionString;

        public NpgsqlDataAccess(string connectionString)
        {
            _connectionString = connectionString;
        }

        // Helper to execute SQL commands that don't return data (INSERT, UPDATE, DELETE)
        private async Task<int> ExecuteNonQueryAsync(string sql, NpgsqlParameter[]? parameters = null)
        {
            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);

            if (parameters != null)
            {
                cmd.Parameters.AddRange(parameters);
            }

            return await cmd.ExecuteNonQueryAsync();
        }
        
        // --- AUTHENTICATION METHODS ---
        
        public async Task<ServiceProvider?> AuthenticateProviderAsync(string username, string password)
        {
            const string sql = @"
                SELECT provider_id, password_hash, full_name, is_admin, assigned_point_id 
                FROM ServiceProviders 
                WHERE username = @Username;";

            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            cmd.Parameters.AddWithValue("@Username", username);

            await using var reader = await cmd.ExecuteReaderAsync();
            if (await reader.ReadAsync())
            {
                var hash = reader.GetString(1);
                // Verify password using BCrypt
                if (BCrypt.Net.BCrypt.Verify(password, hash))
                {
                    return new ServiceProvider
                    {
                        ProviderId = reader.GetInt32(0),
                        PasswordHash = hash,
                        FullName = reader.GetString(2),
                        IsAdmin = reader.GetBoolean(3),
                        AssignedPointId = reader.IsDBNull(4) ? (int?)null : reader.GetInt32(4)
                    };
                }
            }
            return null; // Authentication failed
        }
        
        public async Task UpdateProviderAssignmentAsync(int providerId, int? pointId)
        {
            const string sql = @"
                UPDATE ServiceProviders 
                SET assigned_point_id = @PointId 
                WHERE provider_id = @ProviderId;";
            
            var parameters = new NpgsqlParameter[]
            {
                new("@PointId", pointId.HasValue ? pointId.Value : (object)DBNull.Value),
                new("@ProviderId", providerId)
            };
            await ExecuteNonQueryAsync(sql, parameters);
        }

        // --- CHECK-IN (KIOSK) METHODS ---

        public async Task<List<Service>> GetActiveServicesAsync()
        {
            var services = new List<Service>();
            const string sql = "SELECT service_id, name, description, ticket_prefix FROM Services WHERE is_active = TRUE ORDER BY name;";

            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            
            await using var reader = await cmd.ExecuteReaderAsync();
            while (await reader.ReadAsync())
            {
                services.Add(new Service
                {
                    ServiceId = reader.GetInt32(0),
                    Name = reader.GetString(1),
                    Description = reader.GetString(2),
                    TicketPrefix = reader.GetChar(3)
                });
            }
            return services;
        }

        public async Task<QueueTicket> CreateNewTicketAsync(int serviceId, char prefix)
        {
            QueueTicket newTicket = new();
            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            
            // 1. Determine next sequential number for the prefix in an atomic operation (using transaction)
            await using var transaction = await conn.BeginTransactionAsync();
            
            try
            {
                const string nextNumSql = @"
                    SELECT COALESCE(MAX(CAST(SUBSTRING(ticket_number FROM 2) AS INT)), 0) + 1 
                    FROM QueueTickets 
                    WHERE ticket_number LIKE @Prefix || '%';";
                
                await using var numCmd = new NpgsqlCommand(nextNumSql, conn, transaction);
                numCmd.Parameters.AddWithValue("@Prefix", prefix.ToString());
                var nextNumber = (long)await numCmd.ExecuteScalarAsync();
                var ticketNumber = $"{prefix}{nextNumber:D3}"; 
                
                // 2. Insert the new ticket
                const string insertSql = @"
                    INSERT INTO QueueTickets (ticket_number, service_id, status) 
                    VALUES (@TicketNumber, @ServiceId, 'Waiting') 
                    RETURNING ticket_id, issue_time;";

                await using var insertCmd = new NpgsqlCommand(insertSql, conn, transaction);
                insertCmd.Parameters.AddWithValue("@TicketNumber", ticketNumber);
                insertCmd.Parameters.AddWithValue("@ServiceId", serviceId);

                await using var reader = await insertCmd.ExecuteReaderAsync();
                await reader.ReadAsync();
                
                newTicket = new QueueTicket
                {
                    TicketId = reader.GetInt64(0),
                    TicketNumber = ticketNumber,
                    ServiceId = serviceId,
                    IssueTime = reader.GetDateTime(1),
                    Status = "Waiting"
                };

                await transaction.CommitAsync();
                return newTicket;
            }
            catch (Exception)
            {
                await transaction.RollbackAsync();
                throw;
            }
        }

        // --- SERVICE POINT OPERATIONS ---

        public async Task<QueueTicket?> GetNextNumberAsync(int pointId, int providerId)
        {
            // Tries to find the oldest waiting ticket the point is mapped to, 
            // marks it as 'Called', and assigns the point/provider.
            const string sql = @"
                WITH TargetService AS (
                    SELECT service_id FROM PointServiceMapping WHERE point_id = @PointId
                ),
                CalledTicket AS (
                    SELECT t.ticket_id, t.ticket_number, t.issue_time, s.name as service_name
                    FROM QueueTickets t
                    JOIN Services s ON s.service_id = t.service_id
                    WHERE t.status = 'Waiting' 
                    AND t.service_id IN (SELECT service_id FROM TargetService)
                    ORDER BY t.issue_time ASC
                    LIMIT 1
                    FOR UPDATE SKIP LOCKED -- Critical for multi-provider environment
                )
                UPDATE QueueTickets qt
                SET status = 'Called', 
                    call_time = CURRENT_TIMESTAMP, 
                    service_point_id = @PointId, 
                    service_provider_id = @ProviderId
                FROM CalledTicket ct
                WHERE qt.ticket_id = ct.ticket_id
                RETURNING qt.ticket_id, qt.ticket_number, qt.service_id, ct.service_name;";

            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            cmd.Parameters.AddWithValue("@PointId", pointId);
            cmd.Parameters.AddWithValue("@ProviderId", providerId);

            await using var reader = await cmd.ExecuteReaderAsync();
            if (await reader.ReadAsync())
            {
                return new QueueTicket
                {
                    TicketId = reader.GetInt64(0),
                    TicketNumber = reader.GetString(1),
                    ServiceId = reader.GetInt32(2),
                    ServiceName = reader.GetString(3),
                    ServicePointId = pointId,
                    ServiceProviderId = providerId
                };
            }
            
            return null;
        }

        public Task UpdateTicketStatusAsync(long ticketId, string status)
        {
            string sql = $@"
                UPDATE QueueTickets 
                SET status = @Status, 
                    finish_time = {(status == "Finished" ? "CURRENT_TIMESTAMP" : "NULL")} 
                WHERE ticket_id = @TicketId;";
            
            var parameters = new NpgsqlParameter[]
            {
                new("@Status", status),
                new("@TicketId", ticketId)
            };
            return ExecuteNonQueryAsync(sql, parameters);
        }

        public Task MarkNumberAsNoShowAsync(long ticketId) => UpdateTicketStatusAsync(ticketId, "NoShow");
        public Task MarkNumberAsFinishedAsync(long ticketId) => UpdateTicketStatusAsync(ticketId, "Finished");

        public async Task RecallNumberAsync(long ticketId)
        {
             // Recall means setting status back to 'Called' (or 'Waiting' if needed, but 'Called' is better for flow)
             // We can update the call_time to signal a new call.
            const string sql = @"
                UPDATE QueueTickets 
                SET status = 'Called', call_time = CURRENT_TIMESTAMP, finish_time = NULL
                WHERE ticket_id = @TicketId;";
            
            var parameters = new NpgsqlParameter[]
            {
                new("@TicketId", ticketId)
            };
            await ExecuteNonQueryAsync(sql, parameters);
        }

        public Task TransferNumberAsync(long ticketId, int newPointId)
        {
            const string sql = @"
                UPDATE QueueTickets 
                SET status = 'Waiting', -- Return to waiting pool for the new service point to call
                    transfer_from_point_id = service_point_id,
                    service_point_id = NULL,
                    service_provider_id = NULL,
                    call_time = NULL -- Reset call time
                WHERE ticket_id = @TicketId;";
            
            var parameters = new NpgsqlParameter[]
            {
                new("@TicketId", ticketId)
            };
            return ExecuteNonQueryAsync(sql, parameters);
        }
        
        public async Task<List<QueueTicket>> GetProviderQueueAsync(int pointId)
        {
            var tickets = new List<QueueTicket>();
            const string sql = @"
                SELECT qt.ticket_id, qt.ticket_number, s.name, qt.issue_time, qt.status
                FROM QueueTickets qt
                JOIN Services s ON s.service_id = qt.service_id
                WHERE qt.service_point_id = @PointId AND qt.status IN ('Called', 'Waiting')
                ORDER BY qt.issue_time ASC;";
            
            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            cmd.Parameters.AddWithValue("@PointId", pointId);

            await using var reader = await cmd.ExecuteReaderAsync();
            while (await reader.ReadAsync())
            {
                tickets.Add(new QueueTicket
                {
                    TicketId = reader.GetInt64(0),
                    TicketNumber = reader.GetString(1),
                    ServiceName = reader.GetString(2),
                    IssueTime = reader.GetDateTime(3),
                    Status = reader.GetString(4)
                });
            }
            return tickets;
        }


        // --- WAITING PAGE (DISPLAY) METHOD ---
        
        public async Task<CurrentCallViewModel?> GetLastCalledTicketAsync()
        {
            const string sql = @"
                SELECT qt.ticket_number, sp.name 
                FROM QueueTickets qt
                JOIN ServicePoints sp ON sp.point_id = qt.service_point_id
                WHERE qt.status = 'Called'
                ORDER BY qt.call_time DESC
                LIMIT 1;";

            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            
            await using var reader = await cmd.ExecuteReaderAsync();
            if (await reader.ReadAsync())
            {
                return new CurrentCallViewModel
                {
                    TicketNumber = reader.GetString(0),
                    ServicePointName = reader.GetString(1)
                };
            }
            return null;
        }

        // --- ANALYTICAL REPORTING METHODS ---

        public async Task<List<AnalyticsReportData>> GetServicePointPerformanceReportAsync()
        {
            var reportData = new List<AnalyticsReportData>();
            
            // Calculates Average Waiting Time (IssueTime to CallTime) 
            // and Average Service Time (CallTime to FinishTime) per Service Point.
            const string sql = @"
                SELECT 
                    sp.name AS PointName,
                    COUNT(t.ticket_id) AS CustomersServed,
                    AVG(t.call_time - t.issue_time) AS AvgWaitTime,
                    AVG(t.finish_time - t.call_time) AS AvgServiceTime
                FROM QueueTickets t
                JOIN ServicePoints sp ON sp.point_id = t.service_point_id
                WHERE t.status = 'Finished' AND t.call_time IS NOT NULL AND t.finish_time IS NOT NULL
                GROUP BY sp.name
                ORDER BY sp.name;";

            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);

            await using var reader = await cmd.ExecuteReaderAsync();
            while (await reader.ReadAsync())
            {
                // PostgreSQL returns interval types which Npgsql maps to TimeSpan
                reportData.Add(new AnalyticsReportData
                {
                    Name = reader.GetString(0),
                    CustomersServed = reader.GetInt32(1),
                    AverageWaitingTime = reader.GetFieldValue<TimeSpan>(2),
                    AverageServiceTime = reader.GetFieldValue<TimeSpan>(3)
                });
            }
            
            return reportData;
        }
        
        public async Task<List<AnalyticsReportData>> GetServiceProviderPerformanceReportAsync()
        {
            var reportData = new List<AnalyticsReportData>();
            
            // Calculates performance metrics per Service Provider.
            const string sql = @"
                SELECT 
                    p.full_name AS ProviderName,
                    COUNT(t.ticket_id) AS CustomersServed,
                    AVG(t.call_time - t.issue_time) AS AvgWaitTime,
                    AVG(t.finish_time - t.call_time) AS AvgServiceTime
                FROM QueueTickets t
                JOIN ServiceProviders p ON p.provider_id = t.service_provider_id
                WHERE t.status = 'Finished' AND t.call_time IS NOT NULL AND t.finish_time IS NOT NULL
                GROUP BY p.full_name
                ORDER BY p.full_name;";

            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);

            await using var reader = await cmd.ExecuteReaderAsync();
            while (await reader.ReadAsync())
            {
                reportData.Add(new AnalyticsReportData
                {
                    Name = reader.GetString(0),
                    CustomersServed = reader.GetInt32(1),
                    AverageWaitingTime = reader.GetFieldValue<TimeSpan>(2),
                    AverageServiceTime = reader.GetFieldValue<TimeSpan>(3)
                });
            }
            
            return reportData;
        }


        // --- ADMIN CONFIGURATION METHODS (CRUD Examples) ---

        public async Task<List<ServicePoint>> GetAllServicePointsAsync()
        {
            var points = new List<ServicePoint>();
            const string sql = "SELECT point_id, name, is_active FROM ServicePoints ORDER BY point_id;";

            await using var conn = new NpgsqlConnection(_connectionString);
            await conn.OpenAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            
            await using var reader = await cmd.ExecuteReaderAsync();
            while (await reader.ReadAsync())
            {
                points.Add(new ServicePoint
                {
                    PointId = reader.GetInt32(0),
                    Name = reader.GetString(1),
                    IsActive = reader.GetBoolean(2)
                });
            }
            return points;
        }

        public Task AddServicePointAsync(string name)
        {
            const string sql = "INSERT INTO ServicePoints (name) VALUES (@Name);";
            return ExecuteNonQueryAsync(sql, new NpgsqlParameter[] { new("@Name", name) });
        }
        
        public Task UpdateServicePointAsync(int pointId, string name, bool isActive)
        {
            const string sql = "UPDATE ServicePoints SET name = @Name, is_active = @IsActive WHERE point_id = @PointId;";
            return ExecuteNonQueryAsync(sql, new NpgsqlParameter[] { 
                new("@Name", name), 
                new("@IsActive", isActive),
                new("@PointId", pointId)
            });
        }
    }
}





using QueueManagementSystem.DataAccess;

namespace QueueManagementSystem
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var builder = WebApplication.CreateBuilder(args);

            // Add services to the container.
            builder.Services.AddRazorPages();
            builder.Services.AddControllersWithViews(); // For MVC/API endpoints

            // --- Custom Service Registration ---
            var connectionString = builder.Configuration.GetConnectionString("PostgreSqlConnection");
            
            // Register the custom Data Access Layer as a singleton/scoped service
            // The DAL handles all interaction with the PostgreSQL DB via Npgsql
            builder.Services.AddSingleton(new NpgsqlDataAccess(connectionString));
            // --- End Custom Service Registration ---

            var app = builder.Build();

            // Configure the HTTP request pipeline.
            if (!app.Environment.IsDevelopment())
            {
                app.UseExceptionHandler("/Error");
                app.UseHsts();
            }

            app.UseHttpsRedirection();
            app.UseStaticFiles();

            app.UseRouting();

            app.UseAuthorization();
            
            // Custom routes for the application pages
            app.MapControllerRoute(
                name: "default",
                pattern: "{controller=CheckIn}/{action=Index}/{id?}");

            app.MapRazorPages();

            app.Run();
        }
    }
}