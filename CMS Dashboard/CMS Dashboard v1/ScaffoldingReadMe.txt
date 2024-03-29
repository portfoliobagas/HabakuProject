﻿Scaffolding has generated all the files and added the required dependencies.

However the Application's Startup code may require additional changes for things to work end to end.
Add the following code to the Configure method in your Application's Startup class if not already done:

        app.UseEndpoints(endpoints =>
        {
          endpoints.MapControllerRoute(
            name : "areas",
            pattern : "{area:exists}/{controller=Home}/{action=Index}/{id?}"
          );
        });


        using CMS_Dashboard_v1.Models.ModelModule;
using CMS_Dashboard_v1.Models;
using CMS_Dashboard_v1.Service;
using Microsoft.AspNetCore.Mvc;
using Newtonsoft.Json;
using System.Collections.Generic;
using CMS_Dashboard_v1.Models.ModelForm;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.Rendering;
using System.Reflection;
using System.Net;

namespace CMS_Dashboard_v1.Areas.Form.Controllers
{
    [Area("Form")]
    [Authorize]
    public class MenuController : Controller
    {
        readonly IConfiguration _configuration;
        //BaseResponse<List<MenuModel>> BaseResponse;
        public MenuController(IConfiguration configuration)
        {
            _configuration = configuration;
        }

        public async Task<List<MenuModel>> GetDataApi()
        {
            MasterDataService _masterDataService = new MasterDataService();
            var baseadd = _configuration.GetValue<string>("Api-CMS:BaseAddress");
            var enpoint = _configuration.GetValue<string>("Api-CMS:Menu");
            var data = new BaseResponse<List<MenuModel>>();
            var list = new List<MenuModel>(); 
            var response = await _masterDataService.GetAsync(baseadd + enpoint);

            if (response.IsSuccessStatusCode)
            {
                var jsonString = await response.Content.ReadAsStringAsync();
                data = JsonConvert.DeserializeObject<BaseResponse<List<MenuModel>>>(jsonString);
                list = data.data;
            }

            return list;
        }

        public async Task<IActionResult> Index(MenuModel Model)
        {
            var response =  await GetDataApi();

            try
            {
                if (response != null)
                {
                    Model.List = (from a in response.Where(ss => ss.status)
                                  select new MenuViewModel
                                  {
                                      menu_id = a.menu_id,
                                      menu_name = a.menu_name,
                                  }).ToList();
                }
                else
                {
                    NotifMessage("error", "Gagal load data menu");
                }
            }
            catch (Exception e)
            {
                NotifMessage("error", e.Message.ToString());
            }
            
            return View(Model);
        }

        //private void Dropdown(MenuViewModel Model)
        //{
        //    var general = new GeneralService();
        //    ViewBag.status = new SelectList(general.DropdownStatus(), "text", "text");
        //}

        public void NotifMessage(string status, string messages)
        {
            TempData[status] = status;
            TempData["message"] = messages;
        }

        [Route("Menu/Create")]
        public IActionResult Create()
        {
            var model = new MenuViewModel();
            //Dropdown(model);
            return View();
        }

        [HttpPost]
        [Route("Menu/Create")]
        public async Task<IActionResult> Create(MenuViewModel model)
        {
            var response = await GetDataApi();
            if (response.Any(ss => ss.status && ss.menu_name == model.menu_name))
                ModelState.AddModelError("menu_name", "Nama menu sudah terdaftar");

            try
            {
                if (ModelState.IsValid)
                {
                    var obj = new InsertMenuModel { menu_name = model.menu_name, created_by = HttpContext.Session.GetString("Username")};

                    MasterDataService _masterDataService = new MasterDataService();
                    var baseadd = _configuration.GetValue<string>("Api-CMS:BaseAddress");
                    var enpoint = _configuration.GetValue<string>("Api-CMS:Menu");
                    var data = await _masterDataService.PostAsync(baseadd + enpoint, obj);
                    string jsonString = await data.Content.ReadAsStringAsync();
                    var BaseResponse = JsonConvert.DeserializeObject<BaseResponse<List<MenuModel>>>(jsonString);

                    if(BaseResponse.code == Convert.ToInt16(HttpStatusCode.OK))
                    {
                        NotifMessage("success", "Data berhasil disimpan");
                        return RedirectToAction("Index","Menu");
                    }
                    else
                    {
                        NotifMessage("error", " Data gagal disimpan");
                    }
                }
            }
            catch (Exception e)
            {
                NotifMessage("error", " Error : " + e.Message.ToString());
            }

            //Dropdown(model);
            return View(model);
        }

        public async Task<MenuViewModel> FindData(int id)
        {
            var model = new MenuViewModel();
            var response = await GetDataApi();
            var value = response.Where(ss => ss.menu_id == id && ss.status).FirstOrDefault();

            model.menu_id = value.menu_id;
            model.menu_name = value.menu_name;

            return (model);
        }

        [HttpGet]
        [Route("Menu/Edit")]
        public async Task<IActionResult> Edit(int id)
        {
            var model = await FindData(id);

            return View(model);
        }

        [HttpPost]
        [Route("Menu/Edit")]
        public async Task<IActionResult> Edit(MenuViewModel model)
        {
            var response = await GetDataApi();
            if (response.Any(ss => ss.status && ss.menu_name == model.menu_name && ss.menu_id != model.menu_id))
                ModelState.AddModelError("menu_name", "Nama menu sudah terdaftar");

            try
            {
                if (ModelState.IsValid)
                {
                    var obj = new PutMenuModel {menu_id = model.menu_id, 
                                                menu_name = model.menu_name,
                                                status = true,
                                                modified_by = HttpContext.Session.GetString("Username") };

                    MasterDataService _masterDataService = new MasterDataService();
                    var baseadd = _configuration.GetValue<string>("Api-CMS:BaseAddress");
                    var enpoint = _configuration.GetValue<string>("Api-CMS:Menu");
                    var data = await _masterDataService.PutAsync(baseadd + enpoint, obj);
                    string jsonString = await data.Content.ReadAsStringAsync();
                    var BaseResponse = JsonConvert.DeserializeObject<BaseResponse<List<MenuModel>>>(jsonString);

                    if (BaseResponse.code == Convert.ToInt16(HttpStatusCode.OK))
                    {
                        NotifMessage("success", "Data berhasil diubah");
                        return RedirectToAction("Index", "Menu");
                    }
                    else
                    {
                        NotifMessage("error", " Data gagal diubah");
                    }
                }
            }
            catch (Exception e)
            {
                NotifMessage("error", " Error : " + e.Message.ToString());
            }

            //Dropdown(model);
            return View(model);
        }


        [Route("Menu/Delete")]
        public async Task<IActionResult> Delete(int id)
        {
            var model = await FindData(id);

            return View(model);
        }

        [HttpPost]
        [Route("Menu/Delete")]
        public async Task<IActionResult> Delete(MenuViewModel model)
        {
            try
            {
                if (ModelState.IsValid)
                {
                    var obj = new PutMenuModel
                    {
                        menu_id = model.menu_id,
                        menu_name = model.menu_name,
                        status = false,
                        modified_by = HttpContext.Session.GetString("Username")
                    };

                    MasterDataService _masterDataService = new MasterDataService();
                    var baseadd = _configuration.GetValue<string>("Api-CMS:BaseAddress");
                    var enpoint = _configuration.GetValue<string>("Api-CMS:Menu");
                    var data = await _masterDataService.PutAsync(baseadd + enpoint, obj);
                    var jsonString = await data.Content.ReadAsStringAsync();
                    var BaseResponse = JsonConvert.DeserializeObject<BaseResponse<List<MenuModel>>>(jsonString);

                    if (BaseResponse.code == Convert.ToInt16(HttpStatusCode.OK))
                    {
                        NotifMessage("success", "Data berhasil dihapus");
                        return RedirectToAction("Index", "Menu");
                    }
                    else
                    {
                        NotifMessage("error", " Data gagal dihapus");
                    }
                }
            }
            catch (Exception e)
            {
                NotifMessage("error", " Error : " + e.Message.ToString());
            }

            //Dropdown(model);
            return View(model);
        }

    }
}
