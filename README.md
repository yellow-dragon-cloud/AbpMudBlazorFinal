# MudBlazor Theme in ABP Blazor WebAssembly FINAL

## Introduction

In this part, I will show you how to customize/override pre-built Blazor pages of [Idendity Module](https://docs.abp.io/en/abp/latest/Modules/Identity) using [MudBlazor](https://www.mudblazor.com/) components. I assume that Tenant and Setting management pages are rarely used by end-users, so these modules are not covered in this sample to keep it short. You can customize every page using the principles shown here. All you need to do is:
1. Deriving a class that inherits the class you want to override
2. And adding the following attributes to the derived class:
```razor
@* to the razor file *@
@attribute [ExposeServices(typeof(BASE-CLASS))]
@attribute [Dependency(ReplaceServices = true)]
```
```charp
// or to the C# file
[ExposeServices(typeof(BASE-CLASS))]
[Dependency(ReplaceServices = true)]
```

See [Blazor UI: Customization / Overriding Components](https://docs.abp.io/en/abp/latest/UI/Blazor/Customization-Overriding-Components) for more information.

## 16. Make Sure You Have Completed Previous Parts

You must complete the steps in [Part 1](https://github.com/yellow-dragon-cloud/AbpMudBlazor), [Part 2](https://github.com/yellow-dragon-cloud/AbpMudBlazor2), [Part 3](https://github.com/yellow-dragon-cloud/AbpMudBlazor3), and [Part 4](https://github.com/yellow-dragon-cloud/AbpMudBlazor4) to continue.

## 17. Create Required Files

Create `Identity` folder under `Acme.BookStore.Blazor/Pages/`.

Create the following files in this folder:
- `MudPermissionManagementModal.razor`
- `MudPermissionManagementModal.razor.cs`
- `MudRoleManagement.razor`
- `MudRoleManagement.razor.cs`
- `MudUserManagement.razor`
- `MudUserManagement.razor.cs`

## 18. Modify These Files' Content As Shown Below

<details><summary>`MudPermissionManagementModal.razor`</summary>
<p>
```razor
@using Microsoft.Extensions.Localization
@using Volo.Abp.PermissionManagement.Blazor.Components
@using Volo.Abp.PermissionManagement.Localization
@inherits PermissionManagementModal

<MudDialog @bind-IsVisible="_isVisible"
           Options="DialogOptions">
    <TitleContent>
        <MudText Typo="Typo.h6">                    
            @L["Permissions"] - @_entityDisplayName
        </MudText>
    </TitleContent>
    <DialogContent>
        <MudContainer>
            <MudCheckBox T="bool" @bind-Checked="@GrantAll">
                @L["SelectAllInAllTabs"]
            </MudCheckBox>
            <MudDivider />
            @if (_groups != null)
            {
                <MudTabs Position="MudBlazor.Position.Left" >
                    <TabPanelHeader>
                        <MudText>
                            @* ?! *@
                            &nbsp;&nbsp;&nbsp;&nbsp;
                        </MudText>
                    </TabPanelHeader>
                    <ChildContent>
                        @foreach (var group in _groups)
                        {
                            <MudTabPanel BadgeColor="MudBlazor.Color.Secondary"
                                         BadgeData="@(group.Permissions.Count(x => x.IsGranted))"
                                         Text="@(group.DisplayName)">
                                <MudContainer Style="width: 400px; height: 400px; overflow-y: scroll">
                                    <MudGrid>
                                        <MudItem xs="12">
                                            <MudText Typo="Typo.h6">@group.DisplayName</MudText>
                                            <MudDivider />
                                        </MudItem>
                                        <MudItem xs="12">
                                            <MudCheckBox T="bool"
                                                        Checked="@(group.Permissions.All(x => x.IsGranted))"
                                                        CheckedChanged="@(b => GroupGrantAllChanged(b, group))">
                                            @L["SelectAllInThisTab"]
                                        </MudCheckBox>
                                        <MudDivider />
                                        </MudItem>
                                        @foreach (var permission in group.Permissions)
                                        {
                                            <MudItem xs="1">
                                            </MudItem>
                                            <MudItem xs="11">
                                                <MudCheckBox T="bool"
                                                                Style="padding-inline-start: 16px;"
                                                                Disabled="@(IsDisabledPermission(permission))"
                                                                Checked="@permission.IsGranted"
                                                                CheckedChanged="@(b => PermissionChanged(b, group, permission))">
                                                    @GetShownName(permission)
                                                </MudCheckBox>
                                            </MudItem>
                                        }
                                    </MudGrid>
                                </MudContainer>
                            </MudTabPanel>
                        }
                    </ChildContent>
                </MudTabs>
            }
        </MudContainer>
    </DialogContent>
    <DialogActions>
        <MudButton Color="MudBlazor.Color.Secondary" 
                   OnClick="CloseModalAsync">
            @L["Cancel"]
        </MudButton>
        <MudButton Variant="Variant.Filled" 
                   Color="MudBlazor.Color.Primary" 
                   OnClick="SaveAsync">
            @L["Save"]
        </MudButton>
    </DialogActions>
</MudDialog>
```
</p>
</details>
           
`MudPermissionManagementModal.razor.cs`
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Components;
using MudBlazor;
using Volo.Abp.AspNetCore.Components.Web.Configuration;
using Volo.Abp.PermissionManagement;
using Volo.Abp.PermissionManagement.Localization;

namespace Acme.BookStore.Blazor.Pages.Identity;

public partial class MudPermissionManagementModal
{
    [Inject] private IPermissionAppService PermissionAppService { get; set; }
    [Inject] private ICurrentApplicationConfigurationCacheResetService CurrentApplicationConfigurationCacheResetService { get; set; }

    private bool _isVisible;

    protected virtual DialogOptions DialogOptions
    {
        get => new()
        {
            CloseButton = true,
            CloseOnEscapeKey = true,
            DisableBackdropClick = true,
            MaxWidth = MaxWidth.Medium
        };
    }

    private string _providerName;
    private string _providerKey;

    private string _entityDisplayName;
    private List<PermissionGroupDto> _groups;

    private readonly List<PermissionGrantInfoDto> _disabledPermissions = new();

    private int _grantedPermissionCount = 0;
    private int _notGrantedPermissionCount = 0;

    private bool GrantAll
    {
        get => _notGrantedPermissionCount == 0;
        set
        {
            if (_groups == null) return;

            _grantedPermissionCount = 0;
            _notGrantedPermissionCount = 0;

            foreach (var permission in _groups.SelectMany(x => x.Permissions))
            {
                if (IsDisabledPermission(permission)) continue;

                permission.IsGranted = value;

                if (value)
                    _grantedPermissionCount++;
                else
                    _notGrantedPermissionCount++;
            }
        }
    }

    public MudPermissionManagementModal()
    {
        LocalizationResource = typeof(AbpPermissionManagementResource);
    }

    public async Task OpenDialogAsync(string providerName, string providerKey, string entityDisplayName = null)
    {
        try
        {
            _providerName = providerName;
            _providerKey = providerKey;

            var result = await PermissionAppService.GetAsync(_providerName, _providerKey);

            _entityDisplayName = entityDisplayName ?? result.EntityDisplayName;
            _groups = result.Groups;

            _grantedPermissionCount = 0;
            _notGrantedPermissionCount = 0;
            foreach (var permission in _groups.SelectMany(x => x.Permissions))
            {
                if (permission.IsGranted && permission.GrantedProviders.All(x => x.ProviderName != _providerName))
                {
                    _disabledPermissions.Add(permission);
                    continue;
                }

                if (permission.IsGranted)
                    _grantedPermissionCount++;
                else
                    _notGrantedPermissionCount++;
            }

            _isVisible = true;
            StateHasChanged();
        }
        catch (Exception e)
        {
            await HandleErrorAsync(e);
        }
    }

    private async Task CloseModalAsync()
    {
        await InvokeAsync(() => { _isVisible = false; });
    }

    private async Task SaveAsync()
    {
        try
        {
            var updateDto = new UpdatePermissionsDto
            {
                Permissions = _groups
                    .SelectMany(g => g.Permissions)
                    .Select(p => new UpdatePermissionDto { IsGranted = p.IsGranted, Name = p.Name })
                    .ToArray()
            };

            await PermissionAppService.UpdateAsync(_providerName, _providerKey, updateDto);

            await CurrentApplicationConfigurationCacheResetService.ResetAsync();

            await CloseModalAsync();
        }
        catch (Exception e)
        {
            await HandleErrorAsync(e);
        }
    }

    private void GroupGrantAllChanged(bool value, PermissionGroupDto permissionGroup)
    {
        foreach (var permission in permissionGroup.Permissions)
        {
            if (!IsDisabledPermission(permission))
                SetPermissionGrant(permission, value);
        }
    }

    private void PermissionChanged(bool value, PermissionGroupDto permissionGroup, PermissionGrantInfoDto permission)
    {
        SetPermissionGrant(permission, value);

        if (value && permission.ParentName != null)
        {
            var parentPermission = GetParentPermission(permissionGroup, permission);

            SetPermissionGrant(parentPermission, true);
        }
        else if (value == false)
        {
            var childPermissions = GetChildPermissions(permissionGroup, permission);

            foreach (var childPermission in childPermissions)
            {
                SetPermissionGrant(childPermission, false);
            }
        }
    }

    private void SetPermissionGrant(PermissionGrantInfoDto permission, bool value)
    {
        if (permission.IsGranted == value) return;

        if (value)
        {
            _grantedPermissionCount++;
            _notGrantedPermissionCount--;
        }
        else
        {
            _grantedPermissionCount--;
            _notGrantedPermissionCount++;
        }

        permission.IsGranted = value;
    }

    private static PermissionGrantInfoDto GetParentPermission(PermissionGroupDto permissionGroup, PermissionGrantInfoDto permission)
    {
        return permissionGroup.Permissions.First(x => x.Name == permission.ParentName);
    }

    private static List<PermissionGrantInfoDto> GetChildPermissions(PermissionGroupDto permissionGroup, PermissionGrantInfoDto permission)
    {
        return permissionGroup.Permissions.Where(x => x.Name.StartsWith(permission.Name)).ToList();
    }

    private bool IsDisabledPermission(PermissionGrantInfoDto permissionGrantInfo)
    {
        return _disabledPermissions.Any(x => x == permissionGrantInfo);
    }

    private string GetShownName(PermissionGrantInfoDto permissionGrantInfo)
    {
        if (!IsDisabledPermission(permissionGrantInfo))
            return permissionGrantInfo.DisplayName;

        return string.Format(
            "{0} ({1})",
            permissionGrantInfo.DisplayName,
            permissionGrantInfo.GrantedProviders
                .Where(p => p.ProviderName != _providerName)
                .Select(p => p.ProviderName)
                .JoinAsString(", ")
        );
    }
}
```

`MudRoleManagement.razor`
```razor
@using MudBlazor;
@using Blazorise
@using Blazorise.DataGrid
@using Volo.Abp.BlazoriseUI
@using Volo.Abp.BlazoriseUI.Components
@using Volo.Abp.Identity
@using Microsoft.AspNetCore.Authorization
@using Volo.Abp.PermissionManagement.Blazor.Components
@using Volo.Abp.Identity.Localization
@using Volo.Abp.AspNetCore.Components.Web
@using Volo.Abp.AspNetCore.Components.Web.Theming
@using Volo.Abp.BlazoriseUI.Components.ObjectExtending
@using Volo.Abp.AspNetCore.Components.Web.Theming.Layout
@using Volo.Abp.DependencyInjection
@using Volo.Abp.Identity.Blazor.Pages.Identity
@attribute [Authorize(IdentityPermissions.Roles.Default)]
@attribute [ExposeServices(typeof(RoleManagement))]
@attribute [Dependency(ReplaceServices = true)]
@inject AbpBlazorMessageLocalizerHelper<IdentityResource> LH

@inherits RoleManagement

<MudCard Elevation="8">
    <MudCardHeader>
        @* ************************* PAGE HEADER ************************* *@
        <MudGrid>
            <MudItem>
                <MudText Typo="Typo.h5">@L["Roles"]</MudText>
            </MudItem>
            <MudSpacer />
            <MudItem>
                <MudButton Color="MudBlazor.Color.Primary"
                           Variant="Variant.Outlined"
                           Disabled="!HasCreatePermission"
                           OnClick="OpenCreateModalAsync">
                    @L["NewRole"]
                </MudButton>
            </MudItem>
        </MudGrid>
    </MudCardHeader>
    <MudCardContent>
        @* ************************* DATA GRID ************************* *@
        <MudDataGrid T="IdentityRoleDto"
                     @ref="@_dataGrid"
                     Striped="true"
                     ServerData="LoadServerData">
            <Columns>
                <MudBlazor.Column T="IdentityRoleDto"
                                  Field="@nameof(IdentityRoleDto.Id)"
                                  Title="@L["Actions"]">
                    <CellTemplate>
                        @if (HasUpdatePermission)
                        {
                            <MudIconButton Icon="fas fa-edit" 
                                           OnClick="@(async (_) => { await OpenEditModalAsync(context); })"
                                           Size="MudBlazor.Size.Small" />
                            <MudIconButton Icon="fas fa-user-lock" 
                                           OnClick="@(async (_) => { await OpenPermissionsModalAsync(context); })"
                                           Size="MudBlazor.Size.Small" />
                        }
                        @if (HasDeletePermission)
                        {   
                            <MudIconButton Icon="fas fa-trash" 
                                           OnClick="@(async (_) => { await DeleteEntityAsync(context);} )"
                                           Size="MudBlazor.Size.Small" />
                        }
                    </CellTemplate>
                </MudBlazor.Column>
                <MudBlazor.Column T="IdentityRoleDto"
                                  Field="@nameof(IdentityRoleDto.Name)"
                                  Title=@L["Name"] />
                <MudBlazor.Column T="IdentityRoleDto"
                                  Field="@nameof(IdentityRoleDto.Name)"
                                  Title="">
                    <CellTemplate>
                        @if (context.IsDefault)
                        {
                            <MudChip Color="MudBlazor.Color.Success">
                                @L["DisplayName:IsDefault"]
                            </MudChip>
                        }
                        @if (context.IsPublic)
                        {
                            <MudChip Color="MudBlazor.Color.Info">
                                @L["DisplayName:IsPublic"]
                            </MudChip>
                        }
                    </CellTemplate>
                </MudBlazor.Column>
            </Columns>
        </MudDataGrid>
    </MudCardContent>
</MudCard>

@* ************************* CREATE MODAL ************************* *@
@if (HasCreatePermission)
{
    <MudDialog @bind-IsVisible="_createDialogVisible"
               Options="DialogOptions">
        <TitleContent>
            <MudText Typo="Typo.h6">@L["NewRole"]</MudText>
        </TitleContent>
        <DialogContent>
            <MudForm Model="@NewEntity"
                     @ref="_createForm">
                <MudGrid>
                    <MudItem xs="12">
                        <MudTextField @bind-Value="@NewEntity.Name" 
                                      Label=@L["Name"]
                                      For=@(() => NewEntity.Name) />
                    </MudItem>
                    <MudItem xs="6">
                        <MudCheckBox T="bool" @bind-Checked="@NewEntity.IsDefault">
                            @L["DisplayName:IsDefault"]
                        </MudCheckBox>
                    </MudItem>
                    <MudItem xs="6">
                        <MudCheckBox T="bool" @bind-Checked="@NewEntity.IsPublic">
                            @L["DisplayName:IsPublic"]
                        </MudCheckBox>
                    </MudItem>
                </MudGrid>
            </MudForm>
        </DialogContent>
        <DialogActions>
            <MudButton Color="MudBlazor.Color.Secondary" 
                       OnClick="CloseCreateModalAsync">
                @L["Cancel"]
            </MudButton>
            <MudButton Variant="Variant.Filled" 
                       Color="MudBlazor.Color.Primary" 
                       OnClick="CreateEntityAsync">
                @L["Save"]
            </MudButton>
        </DialogActions>
    </MudDialog>
}

@* ************************* EDIT MODAL ************************* *@
@if (HasUpdatePermission)
{
    <MudDialog @bind-IsVisible="_editDialogVisible"
               Options="DialogOptions">
        <TitleContent>
            <MudText Typo="Typo.h6">@L["Edit"]</MudText>
        </TitleContent>
        <DialogContent>
            <MudForm Model="@EditingEntity"
                     @ref="_editForm">
                <MudGrid>
                    <MudItem xs="12">
                        <MudTextField @bind-Value="@EditingEntity.Name" 
                                      Label=@L["Name"]
                                      For=@(() => EditingEntity.Name) />
                    </MudItem>
                    <MudItem xs="6">
                        <MudCheckBox T="bool" @bind-Checked="@EditingEntity.IsDefault">
                            @L["DisplayName:IsDefault"]
                        </MudCheckBox>
                    </MudItem>
                    <MudItem xs="6">
                        <MudCheckBox T="bool" @bind-Checked="@EditingEntity.IsPublic">
                            @L["DisplayName:IsPublic"]
                        </MudCheckBox>
                    </MudItem>
                </MudGrid>
            </MudForm>
        </DialogContent>
        <DialogActions>
            <MudButton Color="MudBlazor.Color.Secondary" 
                       OnClick="CloseEditModalAsync">
                @L["Cancel"]
            </MudButton>
            <MudButton Variant="Variant.Filled" 
                       Color="MudBlazor.Color.Primary" 
                       OnClick="UpdateEntityAsync">
                @L["Save"]
            </MudButton>
        </DialogActions>
    </MudDialog>
}

@if (HasManagePermissionsPermission)
{
    <MudPermissionManagementModal @ref="_permissionManagementModal"/>
}
```

`MudRoleManagement.razor.cs`
```csharp
using System.Threading.Tasks;
using MudBlazor;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Identity;

namespace Acme.BookStore.Blazor.Pages.Identity;

public partial class MudRoleManagement
{
    private MudForm _createForm;
    private MudForm _editForm;
    private bool _createDialogVisible;
    private bool _editDialogVisible;
    private MudDataGrid<IdentityRoleDto> _dataGrid;
    private MudPermissionManagementModal _permissionManagementModal;

    protected async Task<GridData<IdentityRoleDto>> LoadServerData(GridState<IdentityRoleDto> state)
    {
        if (GetListInput is PagedAndSortedResultRequestDto pageResultRequest)
        {
            pageResultRequest.MaxResultCount = state.PageSize;
            pageResultRequest.SkipCount = state.Page * state.PageSize;
        }
        var result = await AppService.GetListAsync(GetListInput);
        return new()
        {
            Items = result.Items,
            TotalItems = (int)result.TotalCount
        };
    }

    protected override Task OpenCreateModalAsync()
    {
        NewEntity = new();
        _createDialogVisible = true;
        return Task.CompletedTask;
    }

    protected override Task CloseCreateModalAsync()
    {
        _createDialogVisible = false;
        return Task.CompletedTask;
    }

    protected override async Task CreateEntityAsync()
    {
        if (_createForm.IsValid)
        {
            var createDto = MapToCreateInput(NewEntity);
            await AppService.CreateAsync(createDto);
            await _dataGrid.ReloadServerData();
            _createDialogVisible = false;
        }
    }

    protected override Task OpenEditModalAsync(IdentityRoleDto entity)
    {
        EditingEntityId = entity.Id;
        EditingEntity = MapToEditingEntity(entity);
        _editDialogVisible = true;
        return Task.CompletedTask;
    }

    protected override Task CloseEditModalAsync()
    {
        _editDialogVisible = false;
        return Task.CompletedTask;
    }

    protected override async Task UpdateEntityAsync()
    {
        if (_editForm.IsValid)
        {
            await base.UpdateEntityAsync();
            await _dataGrid.ReloadServerData();
            _editDialogVisible = false;
        }
    }

    protected override async Task DeleteEntityAsync(IdentityRoleDto entity)
    {
        if (await Message.Confirm(GetDeleteConfirmationMessage(entity)))
        {
            await base.DeleteEntityAsync(entity);
            await _dataGrid.ReloadServerData();
        }
    }

    protected override Task OnUpdatedEntityAsync()
    {
        return Task.CompletedTask;
    }

    protected virtual async Task OpenPermissionsModalAsync(IdentityRoleDto entity)
    {
        await _permissionManagementModal.OpenDialogAsync(PermissionProviderName, entity.Name);
    }

    protected static DialogOptions DialogOptions
    {
        get => new()
        {
            CloseButton = true,
            CloseOnEscapeKey = true,
            DisableBackdropClick = true
        };
    }
}
```

`MudUserManagement.razor`
```razor
@using Microsoft.AspNetCore.Authorization
@using Volo.Abp.DependencyInjection
@using Volo.Abp.PermissionManagement.Blazor.Components
@using Volo.Abp.Identity.Localization
@using Volo.Abp.AspNetCore.Components.Web.Theming.Layout
@using Volo.Abp.Identity
@using Volo.Abp.Identity.Blazor.Pages.Identity
@attribute [Authorize(IdentityPermissions.Users.Default)]
@attribute [ExposeServices(typeof(UserManagement))]
@attribute [Dependency(ReplaceServices = true)]
@inject AbpBlazorMessageLocalizerHelper<IdentityResource> LH

@inherits UserManagement

<MudCard Elevation="8">
    <MudCardHeader>
        @* ************************* PAGE HEADER ************************* *@
        <MudGrid>
            <MudItem>
                <MudText Typo="Typo.h5">@L["Users"]</MudText>
            </MudItem>
            <MudSpacer />
            <MudItem>
                <MudButton Color="MudBlazor.Color.Primary"
                           Variant="Variant.Outlined"
                           Disabled="!HasCreatePermission"
                           OnClick="OpenCreateModalAsync">
                    @L["NewUser"]
                </MudButton>
            </MudItem>
        </MudGrid>
    </MudCardHeader>
    <MudCardContent>
        @* ************************* DATA GRID ************************* *@
        <MudDataGrid T="IdentityUserDto"
                     @ref="@_dataGrid"
                     Striped="true"
                     ServerData="LoadServerData">
            <Columns>
                <MudBlazor.Column T="IdentityUserDto"
                                  Field="@nameof(IdentityUserDto.Id)"
                                  Title=@L["Actions"]>
                    <CellTemplate>
                        @if (HasUpdatePermission)
                        {
                            <MudIconButton Icon="fas fa-edit"
                                           OnClick="@(async (_) => { await OpenEditModalAsync(context); })"
                                           Size="MudBlazor.Size.Small" />
                            <MudIconButton Icon="fas fa-user-lock"
                                           OnClick="@(async (_) => { await OpenPermissionsModalAsync(context); })"
                                           Size="MudBlazor.Size.Small" />
                        }
                        @if (HasDeletePermission)
                        {
                            <MudIconButton Icon="fas fa-trash"
                                           OnClick="@(async (_) => { await DeleteEntityAsync(context);} )"
                                           Size="MudBlazor.Size.Small" />
                        }
                    </CellTemplate>
                </MudBlazor.Column>
                <MudBlazor.Column T="IdentityUserDto"
                                  Field="@nameof(IdentityUserDto.Name)"
                                  Title=@L["Name"] />
                <MudBlazor.Column T="IdentityUserDto"
                                  Field="@nameof(IdentityUserDto.Email)"
                                  Title=@L["Email"] />
                <MudBlazor.Column T="IdentityUserDto"
                                  Field="@nameof(IdentityUserDto.PhoneNumber)"
                                  Title=@L["PhoneNumber"] />
            </Columns>
        </MudDataGrid>
    </MudCardContent>
</MudCard>

@* ************************* CREATE MODAL ************************* *@
@if (HasCreatePermission)
{
    <MudDialog @bind-IsVisible="_createDialogVisible"
               Options="DialogOptions">
        <TitleContent>
            <MudText Typo="Typo.h6">@L["NewUser"]</MudText>
        </TitleContent>
        <DialogContent>
            <MudForm Model="@NewEntity"
                     @ref="@_createForm">
                <MudContainer>
                    <MudTabs>
                        <MudTabPanel Text=@L["UserInformations"]>
                            <MudContainer Style="width: 450px; height: 450px; overflow-y: scroll">
                                <MudGrid>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@NewEntity.UserName"
                                                      Label=@L["DisplayName:UserName"]
                                                      For="@(() => NewEntity.UserName)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@NewEntity.Name"
                                                      Label=@L["DisplayName:Name"]
                                                      For="@(() => NewEntity.Name)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@NewEntity.Surname"
                                                      Label=@L["DisplayName:Surname"]
                                                      For="@(() => NewEntity.Surname)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@NewEntity.Password"
                                                      Label=@L["DisplayName:Password"]
                                                      InputType="InputType.Password"
                                                      For="@(() => NewEntity.Password)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@NewEntity.Email"
                                                      Label=@L["DisplayName:Email"]
                                                      InputType="InputType.Email"
                                                      For="@(() => NewEntity.Email)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@NewEntity.PhoneNumber"
                                                      Label=@L["DisplayName:PhoneNumber"]
                                                      InputType="InputType.Telephone"
                                                      For="@(() => NewEntity.PhoneNumber)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudCheckBox TValue="bool" 
                                                     @bind-Checked="@NewEntity.IsActive">
                                            @L["DisplayName:IsActive"]
                                        </MudCheckBox>
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudCheckBox TValue="bool" 
                                                     @bind-Checked="@NewEntity.LockoutEnabled">
                                            @L["DisplayName:LockoutEnabled"]
                                        </MudCheckBox>
                                    </MudItem>
                                </MudGrid>
                            </MudContainer>
                        </MudTabPanel>
                        <MudTabPanel Text=@L["Roles"]>
                            <MudContainer Style="width: 450px; height: 450px; overflow-y: scroll">
                                <MudGrid>
                                    @if (NewUserRoles != null)
                                    {
                                        @foreach (var role in NewUserRoles)
                                        {
                                            <MudItem xs="12">
                                                <MudCheckBox TValue="bool" 
                                                             @bind-Checked="@role.IsAssigned">
                                                    @role.Name
                                                </MudCheckBox>
                                            </MudItem>
                                        }
                                    }
                                    else
                                    {
                                        <MudItem xs="12">
                                            <MudText>N/A</MudText>
                                        </MudItem>
                                    }
                                </MudGrid>
                            </MudContainer>
                        </MudTabPanel>
                    </MudTabs>
                </MudContainer>
            </MudForm>
        </DialogContent>
        <DialogActions>
            <MudButton Color="MudBlazor.Color.Secondary" 
                       OnClick="CloseCreateModalAsync">
                @L["Cancel"]
            </MudButton>
            <MudButton Variant="Variant.Filled" 
                       Color="MudBlazor.Color.Primary" 
                       OnClick="CreateEntityAsync">
                @L["Save"]
            </MudButton>
        </DialogActions>
    </MudDialog>
}

@* ************************* EDIT MODAL ************************* *@
@if (HasUpdatePermission)
{
    <MudDialog @bind-IsVisible="_editDialogVisible"
               Options="DialogOptions">
        <TitleContent>
            <MudText Typo="Typo.h6">@L["Edit"]</MudText>
        </TitleContent>
        <DialogContent>
            <MudForm Model="@EditingEntity"
                     @ref="@_editForm">
                <MudContainer>
                    <MudTabs>
                        <MudTabPanel Text=@L["UserInformations"]>
                            <MudContainer Style="width: 450px; height: 450px; overflow-y: scroll">
                                <MudGrid>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@EditingEntity.UserName"
                                                      Label=@L["DisplayName:UserName"]
                                                      For="@(() => EditingEntity.UserName)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@EditingEntity.Name"
                                                      Label=@L["DisplayName:Name"]
                                                      For="@(() => EditingEntity.Name)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@EditingEntity.Surname"
                                                      Label=@L["DisplayName:Surname"]
                                                      For="@(() => EditingEntity.Surname)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@EditingEntity.Password"
                                                      Label=@L["DisplayName:Password"]
                                                      InputType="InputType.Password"
                                                      For="@(() => EditingEntity.Password)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@EditingEntity.Email"
                                                      Label=@L["DisplayName:Email"]
                                                      InputType="InputType.Email"
                                                      For="@(() => EditingEntity.Email)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudTextField @bind-Value="@EditingEntity.PhoneNumber"
                                                      Label=@L["DisplayName:PhoneNumber"]
                                                      InputType="InputType.Telephone"
                                                      For="@(() => EditingEntity.PhoneNumber)" />
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudCheckBox TValue="bool" 
                                                     @bind-Checked="@EditingEntity.IsActive">
                                            @L["DisplayName:IsActive"]
                                        </MudCheckBox>
                                    </MudItem>
                                    <MudItem xs="12">
                                        <MudCheckBox TValue="bool" 
                                                     @bind-Checked="@EditingEntity.LockoutEnabled">
                                            @L["DisplayName:LockoutEnabled"]
                                        </MudCheckBox>
                                    </MudItem>
                                </MudGrid>
                            </MudContainer>
                        </MudTabPanel>
                        <MudTabPanel Text=@L["Roles"]>
                            <MudContainer Style="width: 450px; height: 450px; overflow-y: scroll">
                                <MudGrid>
                                    @if (EditUserRoles != null)
                                    {
                                        @foreach (var role in EditUserRoles)
                                        {
                                            <MudItem xs="12">
                                                <MudCheckBox TValue="bool" 
                                                             @bind-Checked="@role.IsAssigned">
                                                    @role.Name
                                                </MudCheckBox>
                                            </MudItem>
                                        }
                                    }
                                    else
                                    {
                                        <MudItem xs="12">
                                            <MudText>N/A</MudText>
                                        </MudItem>
                                    }
                                </MudGrid>
                            </MudContainer>
                        </MudTabPanel>
                    </MudTabs>
                </MudContainer>
            </MudForm>
        </DialogContent>
        <DialogActions>
            <MudButton Color="MudBlazor.Color.Secondary" 
                       OnClick="CloseEditModalAsync">
                @L["Cancel"]
            </MudButton>
            <MudButton Variant="Variant.Filled" 
                       Color="MudBlazor.Color.Primary" 
                       OnClick="UpdateEntityAsync">
                @L["Save"]
            </MudButton>
        </DialogActions>
    </MudDialog>
}

@if (HasManagePermissionsPermission)
{
    <MudPermissionManagementModal @ref="_permissionManagementModal" />
}
```

`MudUserManagement.razor.cs`
```csharp
using System.Linq;
using System.Threading.Tasks;
using MudBlazor;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Identity;
using Volo.Abp.Identity.Blazor.Pages.Identity;

namespace Acme.BookStore.Blazor.Pages.Identity;

public partial class MudUserManagement
{
    private MudForm _createForm;
    private MudForm _editForm;
    private bool _createDialogVisible;
    private bool _editDialogVisible;
    private MudDataGrid<IdentityUserDto> _dataGrid;
    private MudPermissionManagementModal _permissionManagementModal;

    protected async Task<GridData<IdentityUserDto>> LoadServerData(GridState<IdentityUserDto> state)
    {
        if (GetListInput is PagedAndSortedResultRequestDto pageResultRequest)
        {
            pageResultRequest.MaxResultCount = state.PageSize;
            pageResultRequest.SkipCount = state.Page * state.PageSize;
        }
        var result = await AppService.GetListAsync(GetListInput);
        return new()
        {
            Items = result.Items,
            TotalItems = (int)result.TotalCount
        };
    }

    protected override Task OpenCreateModalAsync()
    {
        NewEntity = new() { IsActive = true };
        NewUserRoles = Roles.Select(i => new AssignedRoleViewModel
        {
            Name = i.Name,
            IsAssigned = i.IsDefault
        }).ToArray();
        _createDialogVisible = true;
        return Task.CompletedTask;
    }

    protected override Task CloseCreateModalAsync()
    {
        _createDialogVisible = false;
        return Task.CompletedTask;
    }

    protected override async Task CreateEntityAsync()
    {
        if (_createForm.IsValid)
        {
            var createDto = MapToCreateInput(NewEntity);
            await AppService.CreateAsync(createDto);
            await _dataGrid.ReloadServerData();
            await CloseCreateModalAsync();
        }
    }

    protected override async Task OpenEditModalAsync(IdentityUserDto entity)
    {
        var userRoleNames = (await AppService.GetRolesAsync(entity.Id)).Items.Select(r => r.Name).ToList();
        EditUserRoles = Roles.Select(x => new AssignedRoleViewModel
        {
            Name = x.Name,
            IsAssigned = userRoleNames.Contains(x.Name)
        }).ToArray();
        EditingEntityId = entity.Id;
        EditingEntity = MapToEditingEntity(entity);
        _editDialogVisible = true;
    }

    protected override Task CloseEditModalAsync()
    {
        _editDialogVisible = false;
        return Task.CompletedTask;
    }

    protected override async Task UpdateEntityAsync()
    {
        if (_editForm.IsValid)
        {
            await AppService.UpdateAsync(EditingEntityId, EditingEntity);
            await _dataGrid.ReloadServerData();
            await CloseEditModalAsync();
        }
    }

    protected override async Task DeleteEntityAsync(IdentityUserDto entity)
    {
        if (await Message.Confirm(GetDeleteConfirmationMessage(entity)))
        {
            await base.DeleteEntityAsync(entity);
            await _dataGrid.ReloadServerData();
        }
    }

    protected override Task OnUpdatedEntityAsync()
    {
        return Task.CompletedTask;
    }

    protected virtual async Task OpenPermissionsModalAsync(IdentityUserDto entity)
    {
        await _permissionManagementModal.OpenDialogAsync(
            PermissionProviderName, entity.Id.ToString());
    }

    protected static DialogOptions DialogOptions
    {
        get => new()
        {
            CloseButton = true,
            CloseOnEscapeKey = true,
            DisableBackdropClick = true
        };
    }
}
```

## Screenshots










