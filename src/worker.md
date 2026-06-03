```bash
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const corsHeaders = {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    };

    if (request.method === "OPTIONS") return new Response(null, { headers: corsHeaders });

    try {
      // ============ 密码验证接口 ============
      if (url.pathname === "/api/verify-password" && request.method === "POST") {
        try {
          const { password } = await request.json();
          
          const expectedPassword = env.ACCESS_PASSWORD || "DEFAULT_ACCESS_PASSWORD_2026";
          
          if (password === expectedPassword) {
            return Response.json({ 
              success: true,
              message: "验证成功"
            }, { headers: corsHeaders });
          } else {
            return Response.json({ 
              success: false,
              message: "密码错误"
            }, { 
              status: 401,
              headers: corsHeaders 
            });
          }
        } catch (error) {
          return new Response(JSON.stringify({ 
            success: false, 
            message: "请求格式错误",
            error: error.message 
          }), {
            status: 400,
            headers: { ...corsHeaders, "Content-Type": "application/json" }
          });
        }
      }

      // ============ 分享接口 ============
      if (url.pathname.startsWith("/api/share/")) {
        const pid = url.pathname.split("/").pop();
        const res = await env.DB.prepare("SELECT content FROM notes WHERE public_id = ? AND is_share_copy = 1").bind(pid).first();
        if (res) {
          await env.DB.prepare("DELETE FROM notes WHERE public_id = ?").bind(pid).run();
          return Response.json(res, { headers: corsHeaders });
        }
        return new Response(JSON.stringify({ error: "失效或已被焚毁" }), { status: 404, headers: corsHeaders });
      }

      // ============ 健康检查接口 ============
      if (url.pathname === "/api/health" && request.method === "GET") {
        return Response.json({ 
          status: "healthy",
          service: "CloudNotes API",
          timestamp: new Date().toISOString(),
          version: "1.0.0"
        }, { headers: corsHeaders });
      }

      // ============ 以下接口需要 ADMIN_KEY 验证 ============
      const auth = request.headers.get("Authorization");
      if (auth !== env.ADMIN_KEY) {
        return new Response(JSON.stringify({ error: "暗号错误" }), { status: 401, headers: corsHeaders });
      }

      // ============ 笔记相关API ============
      if (url.pathname === "/api/save" && request.method === "POST") {
        const { content, public_id, is_share, id } = await request.json();
        
        if (id) {
          await env.DB.prepare(
            "UPDATE notes SET content = ? WHERE id = ?"
          ).bind(content, id).run();
        } else {
          await env.DB.prepare(
            "INSERT INTO notes (content, public_id, is_share_copy) VALUES (?, ?, ?)"
          ).bind(content, public_id || null, is_share ? 1 : 0).run();
        }
        
        return new Response("OK", { headers: corsHeaders });
      }

      if (url.pathname === "/api/list") {
        const { results } = await env.DB.prepare("SELECT * FROM notes WHERE is_share_copy = 0 ORDER BY created_at DESC").all();
        return Response.json(results, { headers: corsHeaders });
      }

      if (url.pathname === "/api/delete" && request.method === "POST") {
        const { id } = await request.json();
        await env.DB.prepare("DELETE FROM notes WHERE id = ?").bind(id).run();
        return new Response("Deleted", { headers: corsHeaders });
      }

      if (url.pathname === "/api/ai-sum" && request.method === "POST") {
        const { text } = await request.json();
        const aiRes = await env.AI.run('@cf/meta/llama-3-8b-instruct', {
          messages: [
            { role: 'system', content: 'You are a helpful assistant. You must summarize the content provided by the user. CRITICAL RULE: If the content is in Chinese, you MUST summarize in Chinese. If the content is in English, you MUST summarize in English.' },
            { role: 'user', content: `Please summarize this:\n${text}` }
          ]
        });
        return Response.json({ summary: aiRes.response }, { headers: corsHeaders });
      }

      // ============ 导航相关API ============
      // 获取导航链接列表
      if (url.pathname === "/api/nav/list" && request.method === "GET") {
        try {
          // 检查nav_links表是否存在
          const { results } = await env.DB.prepare(
            "SELECT * FROM nav_links ORDER BY category, created_at DESC"
          ).all();
          
          return Response.json(results, { headers: corsHeaders });
        } catch (error) {
          // 如果表不存在，返回空数组
          if (error.message.includes("no such table")) {
            return Response.json([], { headers: corsHeaders });
          }
          throw error;
        }
      }

      // 保存/更新导航链接
      if (url.pathname === "/api/nav/save" && request.method === "POST") {
        try {
          const { id, title, url, category, icon, icon_color, description } = await request.json();
          
          if (!title || !url) {
            return new Response(JSON.stringify({ error: "标题和URL不能为空" }), { 
              status: 400, 
              headers: { ...corsHeaders, "Content-Type": "application/json" } 
            });
          }
          
          if (id) {
            // 更新现有链接
            await env.DB.prepare(
              `UPDATE nav_links 
               SET title = ?, url = ?, category = ?, icon = ?, icon_color = ?, description = ?, updated_at = datetime('now')
               WHERE id = ?`
            ).bind(title, url, category || "未分类", icon || "fas fa-globe", icon_color || "icon-blue", description || "", id).run();
          } else {
            // 新建链接
            await env.DB.prepare(
              `INSERT INTO nav_links (title, url, category, icon, icon_color, description, created_at, updated_at)
               VALUES (?, ?, ?, ?, ?, ?, datetime('now'), datetime('now'))`
            ).bind(title, url, category || "未分类", icon || "fas fa-globe", icon_color || "icon-blue", description || "").run();
          }
          
          return new Response("OK", { headers: corsHeaders });
        } catch (error) {
          console.error("保存导航链接失败:", error);
          return new Response(JSON.stringify({ 
            error: "保存失败",
            details: error.message 
          }), {
            status: 500,
            headers: { ...corsHeaders, "Content-Type": "application/json" }
          });
        }
      }

      // 删除导航链接
      if (url.pathname === "/api/nav/delete" && request.method === "POST") {
        try {
          const { id } = await request.json();
          
          if (!id) {
            return new Response(JSON.stringify({ error: "ID不能为空" }), { 
              status: 400, 
              headers: { ...corsHeaders, "Content-Type": "application/json" } 
            });
          }
          
          await env.DB.prepare("DELETE FROM nav_links WHERE id = ?").bind(id).run();
          
          return new Response("Deleted", { headers: corsHeaders });
        } catch (error) {
          console.error("删除导航链接失败:", error);
          return new Response(JSON.stringify({ 
            error: "删除失败",
            details: error.message 
          }), {
            status: 500,
            headers: { ...corsHeaders, "Content-Type": "application/json" }
          });
        }
      }

      // ============ 配置管理API ============
      if (url.pathname === "/api/config/save" && request.method === "POST") {
        const { apiUrl, password, timestamp } = await request.json();
        
        if (!apiUrl || !password) {
          return new Response(JSON.stringify({ error: "缺少必要参数" }), { 
            status: 400, 
            headers: { ...corsHeaders, "Content-Type": "application/json" } 
          });
        }
        
        try {
          await env.DB.prepare(
            "INSERT OR REPLACE INTO system_config (config_key, config_value) VALUES (?, ?)"
          )
            .bind("admin_password", password)
            .run();
          
          return new Response(JSON.stringify({ 
            success: true, 
            message: "配置已保存",
            timestamp: new Date().toISOString() 
          }), { 
            headers: { ...corsHeaders, "Content-Type": "application/json" } 
          });
        } catch (dbError) {
          console.log("system_config表可能不存在，模拟保存成功");
          return new Response(JSON.stringify({ 
            success: true, 
            message: "配置已保存（模拟）",
            timestamp: new Date().toISOString() 
          }), { 
            headers: { ...corsHeaders, "Content-Type": "application/json" } 
          });
        }
      }

      if (url.pathname === "/api/config/get" && request.method === "GET") {
        try {
          const { results } = await env.DB.prepare(
            "SELECT config_key, config_value FROM system_config"
          ).all();
          
          const config = {};
          results.forEach(row => {
            config[row.config_key] = row.config_value;
          });
          
          return Response.json(config, { headers: corsHeaders });
        } catch (error) {
          console.log("system_config表不存在，返回空配置");
          return Response.json({}, { headers: corsHeaders });
        }
      }

      return new Response(JSON.stringify({ error: "Not Found" }), { 
        status: 404, 
        headers: corsHeaders 
      });
    } catch (e) {
      console.error("API Error:", e);
      return new Response(JSON.stringify({ 
        error: e.message,
        stack: e.stack 
      }), { 
        status: 500, 
        headers: corsHeaders 
      });
    }
  }
};
```
